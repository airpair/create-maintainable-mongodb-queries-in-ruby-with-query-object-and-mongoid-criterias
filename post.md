Mongoid is an excellent Ruby ORM for mongodb. Building a mongodb query is pretty simple:

```ruby
Article.where(status: "published", legacy: true, title: /#{keyword}/)
```

You can create criterias inside the `Article` class to promote reusability, ie.

```ruby,linenums=true
# article.rb
def self.published
  where(status: 'published')
end

def self.imported_from_legacy
  where(legacy: true)
end

# refactored query
Article.imported_from_legacy.published.where(title: /#{keyword}/)
```

In a growing application the query conditions are likely to increase in number and complexity and probably need to be driven by some user input.

Following the approach above we'd have:

```ruby,linenums=true
if filter[:published] = 'published'
  criteria = Article.published
elsif filter[:published] = 'unpublished'
  criteria = Article.unpublished
else
  criteria = Mongoid::Criteria.new(Article)
end
criteria.imported_from_legacy.where(title: /#{keyword}/)
```

So far so good. The code deciding which filter to apply is becoming a [transaction script](http://martinfowler.com/eaaCatalog/transactionScript.html), regardless if it lives in a Ruby on Rails controller private method or in a Mongoid Document or in a separate plain Ruby object. A transaction script as defined by Martin Fowler in PEAA:

>>> Organizes business logic by procedures where each procedure handles a single request from the presentation.

Let's extend our example with a couple of features taken from a real application:

* the keyword filter would need to match not only the title but also the article tags.
* let's say we only store the tags ids in the database so we will need to contact the tags API to translate the keyword to an id. 
* the keyword search should be on tags only when a specific string is provided (say "tag:'black'") and ignore titles.

This increase in complexity means the above script needs to have more conditions, broken up in to sub-procedures.

## When to stop using a transaction script

It's hard to tell when to stop on that approach. In general here's a guide line of symptoms you should pay attention to:
 
* adding new filters or changing existing ones becomes hard to estimate
* nested conditions or multiple inline conditions
* test driving is cumbersome, you tend to have one long integration test with a setup populating all the possible combinations
* the code is fragile and breaks often
* the script code becomes a knowledge silos where only one guy approximately knows what's going on and nobody wants to work on it

To me a line count growing over 10/15 is also a signs that we should explore another direction.

## Another approach

What I've been doing when dealing with large more complex queries in the last couple of years is test drive their criterias in separate objects and having a query object orchestrating those filters. I think this as an application of the query object pattern explained by Martin Fowler in PEAA:

>>> A Query Object is an application of the Interpreter pattern geared to represent a SQL query. Its primary roles are to allow a client to form queries of various kinds and to turn those object structures into the appropriate SQL string.

The query is broken up in to criteria objects, and the query object responsibility is in this case just instantiating each criteria with the right parameters and merging them.

When TDD this you'd have a high level integration test testing the first criteria you're adding, then a unit test for the first criteria on the query object, then build that criteria's unit test. At this point your high level integration test should be passing.

How to TDD is not the focus of this post so I omitted the integration test and I am adding the full spec code and class code in one go:

```ruby,linenums=true
describe ContentFilter do

    describe ".apply" do
      let(:criteria) { Mongoid::Criteria.new(Article) }
      let(:keywords) { double }
      let(:published) { double }
      let(:legacy_links) { double }
      let(:piece_type) { double }
      let(:filters) { { keyword: keywords, published: published,
                        legacy_links: legacy_links, piece_type: piece_type } }

      before do
        allow(Criteria::KeywordFilter).to receive(:new).and_return( double(value: criteria) )
        allow(Criteria::PublishedStateFilter).to receive(:new).and_return( double(value: criteria) )
        allow(Criteria::LegacyLinkFilter).to receive(:new).and_return( double(value: criteria) )
        allow(Criteria::PieceTypeFilter).to receive(:new).and_return( double(value: criteria) )
      end

      it "should trigger keyword criteria" do
        keyword_filter = double('KeywordFilter', value: criteria)
        expect(Criteria::KeywordFilter).to receive(:new).with(keywords).and_return(keyword_filter)

        filter = ContentFilter.new(filters)
        filter.apply
      end

      it "should trigger published state criteria" do
        published_filter = double('PublishedStateFilter', value: criteria)
        expect(Criteria::PublishedStateFilter).to receive(:new).with(published).and_return(published_filter)

        filter = ContentFilter.new(filters)
        filter.apply
      end

      it "should trigger legacy link criteria" do
        legacy_links_filter = double('LegacyLinkFilter', value: criteria)
        expect(Criteria::LegacyLinkFilter).to receive(:new).with(legacy_links).and_return(legacy_links_filter)

        filter = ContentFilter.new(filters)
        filter.apply
      end

      it "should trigger piece type criteria" do
        piece_type_filter = double('PieceTypeFilter', value: criteria)
        expect(Criteria::PieceTypeFilter).to receive(:new).with(piece_type).and_return(piece_type_filter)

        filter = ContentFilter.new(filters)
        filter.apply
      end
    end

  end
end
```

This test ensures the correct attribute is passed from the query object (ContentFilter) to each criteria. Each criteria will return a fallback criteria when not used.

And here's the code for it:

```ruby,linenums=true
class ContentFilter

  def initialize(filter_parameters)
    @filter_parameters = filter_parameters || {}
  end

  # Generate a filter on keyword and published state.
  # @return [Mongoid::Criteria]
  def apply
    keyword_criteria.merge(legacy_link_criteria).merge(piece_type_criteria).merge(published_state_criteria)
  end


  private

    def keyword_criteria
      criteria = Criteria::KeywordFilter.new(@filter_parameters[:keyword])
      criteria.value
    end

    def published_state_criteria
      criteria = Criteria::PublishedStateFilter.new(@filter_parameters[:published])
      criteria.value
    end

    def legacy_link_criteria
      criteria = Criteria::LegacyLinkFilter.new(@filter_parameters[:legacy_links])
      criteria.value
    end

    def piece_type_criteria
      criteria = Criteria::PieceTypeFilter.new(@filter_parameters[:piece_type])
      criteria.value
    end
end
```

The logic of the query is broken in small easy to understand parts that when assembled together deliver the full query.

Now let's switch to a unit test for the published filter:

```ruby,linenums=true
describe Criteria::PublishedStateFilter do

  context 'filtering published pieces' do
    let(:published_state) { 'true' }

    it 'should return a Mongoid::Criteria' do
      criteria = Criteria::PublishedStateFilter.new(published_state)
      expect(criteria.value).to be_a_kind_of(Mongoid::Criteria)
    end

    it "should retrieve published pieces" do
      expect(Article).to receive(:published)
      
      filter = Criteria::PublishedStateFilter.new(published_state)
      filter.value
    end
  end

  context 'filtering unpublished pieces' do
    let(:published_state) { 'false' }

    it "should retrieve unpublished pieces" do
      expect(Article).to receive(:unpublished)
      
      filter = Criteria::PublishedStateFilter.new(published_state)
      filter.value
    end
  end

  context 'without a published state filter' do
    let(:published_state) { nil }

    it "should apply no filter" do
      expect(Article).to_not receive(:published)
      expect(Article).to_not receive(:unpublished)
      
      filter = Criteria::PublishedStateFilter.new(published_state)
      filter.value
    end
  end

end
```

Our unit test is ensuring that based on user input the correct mongoid API are made. This object's value will always be a `Mongoid::Criteria` to allow the `ContentFilter` class to merge it with other criterias.

Here's the code:

```ruby,linenums=true
module Criteria

  class PublishedStateFilter

    def initialize(state_filter_parameter)
      @published_state = state_filter_parameter
    end

    # Generate a filter on published state.
    # @return [Mongoid::Criteria]
    def value
      if filtering_published?
        Article.published
      elsif filtering_unpublished?
        Article.unpublished
      else
        Mongoid::Criteria.new(Article)
      end
    end


    private

      def filtering_published?
        @published_state == 'true'
      end

      def filtering_unpublished?
        @published_state == 'false'
      end

  end

end
```

Now let's look at the keyword filter, it has to work in two scenarios: a tag specific filter ie. `"tag:'president day'"` or `"tag:'public holiday'"` (to find any article tagged with the text between quotes) as well as: "president" to find articles titled and tagged "president". That logic will live inside the `Criteria::KeywordFilter`.

Now we start building the keyword criteria test:

```ruby,linenums=true
describe Criteria::KeywordFilter do

  context "with a keyword filter" do
    let(:keyword) { 'president' }
    let(:president) { "ed723f60-afce-11e4-ab7d-12e3f512a338"  }
    let(:president_obama) { "ed7241f4-afce-11e4-ab7d-12e3f512a338" }
    before do
      # Here we are stubbing our Api interface to say when I search for 'president'
      # return two terms: president and president obama
      allow_any_instance_of(TermsApi::Searcher).to receive(:match).with(keyword).and_return([president, president_obama])
    end

    it 'should generate a criteria' do
      criteria = Criteria::KeywordFilter.new(keyword)
      expect(criteria.value).to be_kind_of(Mongoid::Criteria)
    end

    it 'should match headline and tags' do
      expect(Article).to receive(:or).with( [{ term_ids: { "$in" => [president, president_obama] } },
                                             { headline: /#{keyword}/i },
                                            ])

      criteria = Criteria::KeywordFilter.new(keyword)
      criteria.value
    end
  end


  context "with an exact keyword filter on tag" do
    let(:tag) { "Music Ceremony" }
    let(:keyword) { %Q{tag:'#{tag}'} }
    let(:president) { "ed723f60-afce-11e4-ab7d-12e3f512a338"  }
    let(:president_obama) { "ed7241f4-afce-11e4-ab7d-12e3f512a338" }
    before do
      # Here we are stubbing our Api interface to say when I search for the exact term 'Music Ceremony'
      # return only that term.
      expect_any_instance_of(TermsApi::Searcher).to receive(:match_exactly).with(tag).and_return([president])
    end

    it 'should match only for the exact tag' do
      expect(Article).to receive(:in).with(term_ids: [president])

      criteria = Criteria::KeywordFilter.new(keyword)
      criteria.value
    end
  end

end
```

Our unit test is ensuring that based on keyword filters the correct mongoid API as well as 3rd party calls for terms are made. Like for `PublishedFilter` this object's value will always be an instance of `Mongoid::Criteria` to allow the `ContentFilter` class to merge it.

I once again pasted the entire spec but while TDD-ing you'd add one example at the time. Based on how critical the feature is you might start from a specific integration test.

And here's the code:

```ruby,linenums=true
module Criteria

  class KeywordFilter

    # Generate a filter on keyword matching headline, term or exact term.
    def initialize(keyword)
      @keyword = keyword
    end

    # @return [Mongoid::Criteria]
    def value
      if filtering_by_exact_term?
        Article.in( term_ids: exact_term_to_ids )
      elsif filtering_by_keyword?
        Article.or(matching_headline_or_terms)
      else 
        Mongoid::Criteria.new(Article)
      end
    end


    private
      
      def filtering_by_exact_term?
        filtering_by_keyword? &&
        @keyword.match(/^tag:'.+'$/)
      end
      
      def exact_term_to_ids
        searcher = TermsApi::Searcher.new
        searcher.match_exactly(extract_exact_term_from_keyword)
      end
      
      def extract_exact_term_from_keyword
        @keyword.match(/tag:'(.+)'/)
        $1
      end
      
      def filtering_by_keyword?
        @keyword.present?
      end
      
      def matching_headline_or_terms
        [ { term_ids: { "$in" => term_to_ids } },
          { headline: /#{@keyword}/i } ]
      end

      def term_to_ids
        searcher = TermsApi::Searcher.new
        searcher.match(@keyword)
      end

  end
end
```

We're using a higher level language to represent the query composition which helps to better understand what the code does.

## Conclusion

I find this approach useful to organize complex query code and to steadily expand queries as an application grows.

Failing to see that growth and continuing to extend a transaction script will lead to your domain model complexity be tangled in code making it hard to maintain.

You can add sorting capabilities with a `ContentSorter` class that given the `ContentFilter` applies the necessary sorting logic.