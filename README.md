Deprecated
==========

This modified version of the gem is no longer necessary, as [the master repository](https://github.com/artsy/mongoid_fulltext)
has been adjusted to be compatible with `Ruby 1.8.x`.

---

Mongoid Fulltext Search
=======================

Full-text search using n-gram matching for the Mongoid ODM. Tested on MongoDB 1.6 and above, but
probably works on earlier versions as well.

This Version
------------

This fork removes the dependency on the `unicode_utils` gem, and thus makes it compatible with Ruby 1.8.
It may not escape/remove accented characters quite as well as the original but it is a solution for those
that need compatibility with `Ruby 1.8.x`.

Some examples:
--------------
    
Suppose you have an `Artist` model and want to index each artist's name:

    class Artist
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :first_name
      field :last_name

      def name
        [first_name, last_name].join(' ')
      end

      fulltext_search_in :name
    end

The `fulltext_search_in` directive will index the full name of the artist, so now
you can call:

    Artist.fulltext_search("vince vangogh")

which will return an array of the Artist instances that best match the search string. Most likely,
Vincent van Gogh will be included in the results. You can index multiple fields with the same
index, so we can get the same effect of our Artist index above using:

    class Artist
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :first_name
      field :last_name

      fulltext_search_in :first_name, :last_name
    end

To restrict the number of results returned, pass the `:max_results` parameter to `fulltext_search`:

    Artist.fulltext_search("vince vangogh", { :max_results => 5 })

To return a pair of `[ result, score ]` instead of an array of results, pass the `:return_scores` parameter to `fulltext_search`:

    Artist.fulltext_search("vince vangogh", { :return_scores => true })

The larger a score is, the better mongoid_fulltext thinks the match is. The scores have the following rough 
interpretation that you can use to make decisions about whether the match is good enough:

* If a prefix of your query matches something indexed, or if your query matches a prefix of something
  indexed (for example, searching for "foo" finds "myfoo" or searching for "myfoo" finds "foo"), you
  can expect a score of at least 1 for the match.
* If an entire word in your query matches an entire word that's indexed and you have the `index_full_words`
  option turned on (it's turned on by default), you can expect a score of at least 2 for the match.
* If neither of the above criteria are met, you can expect a score less than one.

If you don't specify a field to index, the default is the result of `to_s` called on the object.
The following definition will index the first and last name of an artist:

    class Artist
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :first_name
      field :last_name

      def to_s
        '%s %s' % [first_name, last_name]
      end

      fulltext_search_in
    end 

The full-text index is stored in a separate MongoDB collection in the same database as the
models you're indexing. By default, the name of this collection is generated for you. Above,
a collection named something like `mongoid_fulltext.index_artist_0` will be created to
hold the index data. You can override this naming and provide your own collection name with 
the :index_name parameter:

    class Artwork
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :title
      fulltext_search_in :title, :index_name => 'mongoid_fulltext.foobar'
    end

You can also create multiple indexes on a single model, in which case you'll want to
provide index names:

    class Artwork
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :title
      field :artist_name
      field :gallery_name
      filed :gallery_address

      fulltext_search_in :title, :index_name => 'title_index'
      fulltext_search_in :artist_name, :index_name => 'artist_name_index'
      fulltext_search_in :gallery_name, :gallery_address, :index_name => 'gallery_index'
    end

The index names are helpful now because you'll have to specify which one you want to use when you
call `fulltext_search`:

    Artwork.fulltext_search('warhol', :index => 'artist_name_index')

If you have multiple indexes specified and you don't supply a name to `fulltext_search`, the
method call will raise an exception.

If you're indexing multiple models, you may find that you need to combine results to create
a single result set. For example, if both the `Artist` model and the `Artwork` model are
indexed for full-text search, then to get results from both, you'd have to call 
`Artist.fulltext_search` and `Artwork.fulltext_search` and combine the results yourself. If
your intention is instead to get the top k results from both Artists and Artworks, you can
merge both into a single index by using the same `:external_index` parameter:

    class Artwork
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :title
      fulltext_search_in :title, :index_name => 'artwork_and_artists'
    end

    class Artist
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :name
      fulltext_search_in :name, :index_name => 'artwork_and_artists'
    end

Now that these two models share the same external index collection, we can search them both through
either model's `fulltext_search` method:

    Artwork.fulltext_search('picasso')  # returns same results as Artist.fulltext_search('picasso')

If you want to filter the results from full-text search, you set up filters when the indexes are
defined. For example, suppose that in addition to wanting to use the `artwork_and_artists` index
defined above to search for `Artwork`s or `Artist`s, we want to be able to run full-text searches
for artists only and for artworks priced above $10,000. Instead of creating two new indexes or
attempting to filter the results after the query is run, we can specify the filter predicates
at the time of index definition:

    class Artwork
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :title
      field :price
      fulltext_search_in :title, :index_name => 'artwork_and_artists',
                         :filters => { :is_expensive => lambda { |x| x.price > 10000 },
                                       :has_long_name => lambda { |x| x.title.length > 20 }}
    end

    class Artist
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :name
      field :birth_year
      fulltext_search_in :name, :index_name => 'artwork_and_artists',
                         :filters => { :born_before_1900 => lambda { |x| x.birth_year < 1900 },
                                       :has_long_name => lambda { |x| x.name.length > 20}}
    end

After defining filters, you can query for results that match particular values of filters:

    # Only return artists born before 1900 that match 'foobar'
    Artist.fulltext_search('foobar', :born_before_1900 => true)

    # Return artists or artworks that match 'foobar' and have short names
    Artist.fulltext_search('foobar', :has_long_name => false)

    # Only return artworks with prices over 10000 that match 'mona lisa'
    Artwork.fulltext_search('mona lisa', :is_expensive => true)

    # Only return artworks with prices less than 10000 that match 'mona lisa'
    Artwork.fulltext_search('mona lisa', :is_expensive => false)

Note that in all of the example queries above, supplying a filter that is defined on exactly 
one of the models will restrict the search to results from that model only. For example,
since `:is_expensive` is defined only on `Artwork`s, a call to `fulltext_search` with either
`:is_expensive => true` or `:is_expensive => false` will return only `Artwork` results.

You can specify multiple filters per index and per model. Each filter is a predicate that will 
be called on objects as they're inserted into the full-text index (any time the model is saved.) 
Filters are only called on instances of models they're defined on, so in the example above, the 
`is_expensive` filter is only applied to instances of `Artwork` and the `born_before_1900` filter 
is only applied to instances of `Artist`, although both filters can be used when querying from 
either model. The `has_long_name` filter, on the other hand, will return instances of both 
`Artwork` and `Artist` since it's defined on each model.

Filters shouldn't ever throw, but if they do, the filter is just ignored. If you apply filters to
indexes that are on multiple fields, the filter is applied to each field and the filter result is
the AND of all of the individual results for each of the fields. Finally, if a filter is defined 
but criteria for that filter aren't passed to `fulltext_search`, the result is as if the filter 
had never been defined - you see both models that both pass and fail the filter in the results.

Indexing Options
----------------

Additional indexing/query options can be used as parameters to `fulltext_search_in`.

* `alphabet`: letters to index, default is `abcdefghijklmnopqrstuvwxyz0123456789 `
* `word_separators`: word separators, default is the space character.
* `ngram_width`: ngram width, default is `3`
* `index_full_words`: index full words, which improves exact matches, default is `true`.
* `index_short_prefixes`: index a prefix of each full word of length `(ngram_width-1)`. Useful if
  you use a larger ngram_width than the default of 3. Default for this option is `false`.
* `stop_words`: a hash of words to avoid indexing as full words. Used only if `index_full_words`
  is set to `true`. Defaults to a hash containing a list of common English stop words.
* `apply_prefix_scoring_to_all_words`: score n-grams at beginning of words higher, default is `true`
* `max_ngrams_to_search`: maximum number of ngrams to query at any given time, default is `6`
* `max_candidate_set_size`: maximum number of candidate ngrams to examine for a given query.
  Defaults to 1000. If you're seeing poor results, you can try increasing this value to consider
  more ngrams per query (changing this parameter does not require a re-index.) The amount of time
  a search takes is directly proportional to this parameter's value.
* `remove_accents`: remove accents on accented characters or not. Defaults to true.  If a string 
  is encoded in UTF-8, we strip the accents using NFKD normalization (via an external library, 
  `unicode_utils`. If a string is encoded in ASCII-8BIT, we assume it has been passed via a
  URL, for instance we might have "%C3%A9" which is how an "e-accute" ("é") gets passed
  through a web-browser. These are then changed to their UTF-8 equivalents (via the `CGI` gem) 
  and then finally stripped, as before.
* `update_if`: controls whether or not the index will be updated. This can be set to a symbol,
  string, or proc. If the result of evaluating the value is true, the index will be updated.
    * When set to a symbol, the symbol is sent to the document.
    * When set to a string, the string is evaluated within the document's instance.
    * When set to a proc, the proc is called, and the document is given to the proc as the first arg.
    * When set to any other type of object, the document's index will not be updated.

Array filters
-------------

A filter may also return an Array. Consider the following example.

    class Artist
      include Mongoid::Document
      include Mongoid::FullTextSearch

      field :name
      field :exhibitions, as: Array, default: []

      fulltext_search_in :name, :index_name => 'exhibited_artist',
        :filters => { 
          :exhibitions => lambda { |artist| artist.exhibitions }
        }
    end

You can now find all artists that are at the Art Basel exhibition or all artists that have exhibited
at both the Art Basel and the New York Armory exhibition.

    # All artists
    Artist.fulltext_search('foobar')

    # Artists at the Art Basel exhibition only
    Artist.fulltext_search('foobar', :exhibitions => [ "Art Basel" ])

    # Artists at both the Art Basel and the New York Armory exhibition
    Artist.fulltext_search('foobar', :exhibitions => [ "Art Basel", "New York Armory" ])

Building the index
------------------

The fulltext index is built and maintained incrementally by hooking into `before_save` and 
`before_destroy` callbacks on each model that's being indexed. If you want to build an index
on existing models, you can call the `update_ngram_index` method on the class or each instance:

    Artwork.update_ngram_index
    Artwork.find(id).update_ngram_index

You can remove all or individual instances from the index with the `remove_from_ngram_index`
method:

    Artwork.remove_from_ngram_index
    Artwork.find(id).remove_from_ngram_index

The methods on the model level perform bulk removal operations and are therefore faster that 
updating or removing records individually.

If you need to control when the index is updated, provide the `update_if` option to
`fulltext_search_in`, and set it to a symbol, string, or proc. Eg:

    # Only update the "age" index if the "age" field has changed.
    fulltext_search_in :age,    :update_if => :age_changed?

    # Only update the "names" index if the "firstname" or "lastname" field has changed.
    fulltext_search_in :names,  :update_if => "firstname_changed? || lastname_changed?"

    # Only update the "email" index if the "email" field ends with "gmail.com".
    fulltext_search_in :email,  :update_if => Proc.new { |doc| doc.email.match /gmail.com\Z/ }

Mongo Database Indexes
----------------------

Mongoid provides an indexing mechanism on its models triggered by the `create_indexes` method.
Mongoid_fulltext will hook into that behavior and create appropriate database indexes on its
collections. These indexes are required for an efficient full text search. 

Creating database indexes is typically done with the `db:mongoid:create_indexes` task.

    rake db:mongoid:create_indexes

Running the specs
-----------------

To run the specs, execute `rake spec`. You need a local MongoDB instance to run the specs.

