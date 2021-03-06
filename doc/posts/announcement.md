
# Announcing servant-quickcheck

Some time ago, we released `servant-mock`. The idea behind it is to use
`QuickCheck` to create a mock server that accords with a servant API. Not long
after, we started thinking about an analog that would, instead of mocking a
server, mock a client instead - i.e., generate random requests that conform to
an API description.

This is much closer to the traditional use of `QuickCheck`. The most obvious
use-case is checking that properties hold of an *entire* server rather than of
individual endpoints. (But there are other uses that you can skip to if they
sound more interesting.)

## `serverSatisfies`

A useful guideline when writing and maintaing software is that, if there isn't
a test for a behaviour or property, sooner or later that property will be broken.
Another important perspective is that tests are a form of documentation - the
present developer telling future ones "this matters, and should be this way".

The advantage of using tests for this form of documentation is that there's
simply too much information to convey, some of it only relevant to very specific
use cases, and rather than overload developers with an inexhaustible quantity of
details that would be hard to keep track of or remember, tests are a mechanism
of reminding developers of *only the relevant information, at the right time*.
<<EXAMPLE>>.

We might hope that we could use tests to communicate the wide array of best
practices that have developed around APIs. About to return a top-level integer
in JSON? A test should say that's bad practice. About to not catch exceptions
and give a more meaningful HTTP status code? Another test there to stop you.

Traditionally, in web services these things get done at the level of *individual*
endpoints. But this means that if a developer who hasn't had extensive experience with web
programming best practices writes a *new* endpoint which *does* return a top-level
integer literal, there's no test there to stop her. Code review might help, but
code review is much more error prone than tests, and really only meant for those
things that are too subtle to automate. (Indeed, if code review were such a reliable
defense mechanism against bugs and bad code, why have tests and linters at all?)

The problem, then, with thinking about tests as only existing at the level of individual
endpoints is that there are no tests *for* tests - tests that check that new
behaviour and tests conforms to higher-level, more general best practices.

`servant-quickcheck` aims to solve that. It allows describing properties that
*all* endpoints must satisfy. If a new endpoint comes along, it too will be
tested for that property, without any further work.

Why isn't this idea already popular? Well, most web frameworks don't have a
reified description of APIs (beyond perhaps the routes). When you don't know
what the endpoints of an application are, and what request body they expect,
trying to generate arbitrary requests is almost entirely going to result in
404s (not found) and 400s (bad request). Maybe one in a thousand requests will
actually test a handler. Not very useful.

`servant` applications, on the other hand, have a machine-readable API description
already available. And they already associate "correct" requests with particular
types. It's a small step, therefore, to generate 'arbitrary' values for these
requests, and all of them will go through to your handlers. (Note: all of the
uses of `servant-quickcheck` work with applications *not* written with servant-server -
and indeed not *in Haskell - but the API must be described with the servant
DSL.)

Let's see how this works in practice.  As a running example, let's use a simple
service that allows adding, removing, and querying biological species. Our SQL
schema is:


> **schema.sql**

>     
>     CREATE TABLE genus (
>         genus_name     text  PRIMARY KEY,
>         genus_family   text  NOT NULL
>     );
>     
>     CREATE TABLE species (
>         species_name    text  PRIMARY KEY,
>         species_genus   text  NOT NULL REFERENCES genus (genus_name)
>     )


And our actual application:


> **Main.hs**

>     {-# LANGUAGE DataKinds #-}
>     {-# LANGUAGE DeriveAnyClass #-}
>     {-# LANGUAGE DeriveGeneric #-}
>     {-# LANGUAGE TypeOperators #-}
>     {-# LANGUAGE OverloadedStrings #-}
>     {-# LANGUAGE RecordWildCards #-}
>     module Main where
>     
>     import Servant
>     import Data.Aeson
>     import Database.PostgreSQL.Simple
>     import GHC.Generics (Generic)
>     import Data.Text (Text)
>     import Network.Wai.Handler.Warp
>     import Control.Monad.IO.Class (liftIO)
>     
>     type API
>       = "species" :> (Capture "species-name" Text :> ( Get '[JSON] Species
>                                                   :<|> Delete '[JSON] ())
>                  :<|> ReqBody '[JSON] Species :> Post '[JSON] ())
>       -- The plural of 'species' is unfortunately also 'species'
>      :<|> "speciess" :> Get '[JSON] [Species]
>     
>     api :: Proxy API
>     api = Proxy
>     
>     data Species = Species
>       { speciesName  :: Text
>       , speciesGenus :: Text
>       } deriving (Eq, Show, Read, Generic, ToJSON, FromJSON)
>     
>     data Genus = Genus
>       { genusName   :: Text
>       , genusFamily :: Text
>       } deriving (Eq, Show, Read, Generic, ToJSON, FromJSON)
>     
>     instance FromRow Genus
>     instance FromRow Species
>     
>     server :: Connection -> Server API
>     server conn = ((\sname -> liftIO (lookupSpecies conn sname)
>                          :<|> liftIO (deleteSpecies conn sname))
>              :<|> (\species -> liftIO $ insertSpecies conn species))
>              :<|> (liftIO $ allSpecies conn)
>     
>     lookupSpecies :: Connection -> Text -> IO Species
>     lookupSpecies conn name = do
>       [s] <- query conn "SELECT * FROM species WHERE species_name = ?" (Only name)
>       return s
>     
>     deleteSpecies :: Connection -> Text -> IO ()
>     deleteSpecies conn name = do
>       _ <- execute conn "DELETE FROM species WHERE species_name = ?" (Only name)
>       return ()
>     
>     insertSpecies :: Connection -> Species -> IO ()
>     insertSpecies conn Species{..} = do
>       _ <- execute conn "INSERT INTO species (species_name, species_genus) VALUES (?)" (speciesName, speciesGenus)
>       return ()
>     
>     allSpecies :: Connection -> IO [Species]
>     allSpecies conn = do
>       query_ conn "SELECT * FROM species"
>     
>     main :: IO ()
>     main = do
>       conn <- connectPostgreSQL "dbname=servant-quickcheck"
>       run 8090 (serve api $ server conn)


(You'll also also need to run:

```
createdb servant-quickcheck
psql --file schema.sql -d servant-quickcheck
```

If you want to run this example.)

This is a plausible effort. You might want to spend a moment thinking about what
could be improved.


> **Spec.hs**

>     
>     {-# LANGUAGE OverloadedStrings #-}
>     module Spec (main) where
>     
>     import Main (server, api, Species(..))
>     import Test.Hspec
>     import Test.QuickCheck.Instances
>     import Servant.QuickCheck
>     import Test.QuickCheck (Arbitrary(..))
>     import Database.PostgreSQL.Simple (connectPostgreSQL)
>     
>     spec :: Spec
>     spec = describe "the species application" $ beforeAll check $ do
>       let pserver = do
>             conn <- connectPostgreSQL "dbname=servant-quickcheck"
>             return $ server conn
>     
>     
>       it "should not return 500s" $ do
>     
>       it "should not return 500s" $ do
>         withServantServer api pserver $ \url ->
>           serverSatisfies api url defaultArgs (not500 <%> mempty)
>     
>       it "should not return top-level json" $ do
>         withServantServer api pserver $ \url ->
>           serverSatisfies api url defaultArgs (onlyJsonObjects <%> mempty)
>     
>       where
>         check = do
>           mvar  <- newMVar []
>           withServantServer api pserver $ \url ->
>              serverSatisfies api url defaultArgs (onlyJsonObjects <%> mempty)
>     
>     main :: IO ()
>     main = do
>       hspec spec
>     
>     instance Arbitrary Species where
>       arbitrary = Species <$> arbitrary <*> arbitrary


But this fails in quite a few ways.


<<TODO>>

This was an example created with the knowledge of what it was supposed to
exemplify. To try to get a more accurate assessment of the practical usefulness
of `servant-quickcheck`, I tried running `serverSatisfies` with a few
predicates over some of the open-source `servant` servers I could find, and
results were also promising.

There are probably a lot of other interesting properties that one might to add
besides those I've included. As an example, we could have a property that
all HTML is checked against, which is sometimes tricky for HTML that's
generated dynamically. Or check that every page has a Portuguese translation.

### Why best practices are good

As a side note: you might have wondered "why bother with API best practices?".
It is, it has to be said, a lot of extra (as in not only getting the feature done)
work to do, for dubious benefit. And indeed, the relevance of discoverability, for
example, unclear, since not that many tools use it as perhaps was anticipated.

But `servant-quickcheck` both makes it *easier* to conform to best practices,
and exemplifies their advantage in enabling better tooling. If we pick 201 (Success, the 'resource' was
created), rather than the more generic 200 (Success), and do a *little* more work
by knowing to make this decision, `servant-quickcheck` knows this means there
should be some representation of the resource created. So it knows to ask you
for a link to it (the RFC creators thought to ask for this). And if you do (again,
a little more work), `servant-quickcheck` will know to try to look at that
resource by following the link, checking that it's not broken, and maybe even
returns a response that equivalent to the original POST request). And then it
finds a real bug - your application allows species with '/' in their name to
be created, but not queried with a 'GET' for! This, I think, is already a win.


## `serversEqual`

There's another very appealing application of the ability to generate "sensible"
arbitrary requests. It's for testing that two applications are equal. We can generate arbitrary
requests, send them to both servers (in the same order), and check that the responses
are equivalent.  (This was, incidentally, one of the first applications of
`servant-client`, albeit in a much more manual way, when we rewrote a microservice
originally in Python in Haskell.) Generally with rewrites, even if there's some
behaviour that isn't optimal, perhaps a lot of things already depend on that service
and make interace poorly with "improvements", so it makes sense to first mimick
*exactly* the original behaviour, and only then aim for improvements.

`servant-quickcheck` provides a single function, `serversEqual`, that attempts
to verify the equivalence of servers. Since some aspects of responses might not
be relevant (for example, whether the the `Server` header is the same, or whether
two JSON responses have the same formatting), it allows you to provide a custom
equivalence function. Other than that, you need only provide an API type and two
URLs for testing, and the rest `serversEqual` handles.

## Future directions: benchmarking

What else could benefit from tooling that can automatically generate sensible
(*vis-a-vis* a particular application's expectations) requests?

One area is extensive automatic benchmarking. Currently we use tools such as
`ab`, `wrk`, `httperf` in a very manual way - we pick a particular request that
we are interested in, and write a request that gets made thousands of times.
But now we can have a multiplicity of requests to benchmark with! This allows
*finding* slow endpoints, as well as (I would imagine, though I haven't actually
tried this yet) finding synchronization issues that make threads wait for too
long (such as waiting on an MVar that's not really needed), bad asymptotics
with respect to some other type of request.

(On this last point, imagine not having an index in a database for "people",
 and having a tool that discovers that the latency on a search by first name
 grows linearly with the number of POST requests to a *different* endpoint! We'd
 need to do some work to do this well, possibly involving some machine
 learning, but it's an interesting and probably useful idea.)


# Conclusion

I hope this library presents some useful functionality already, but I hope
you'll also think how it could be improved!

There'll be a few more packages in the comings weeks - check back soon!

**Note**: This post is an anansi literate file that generates multiple source
files. They are:


> ** Main.hs**

>     Main.hs



> ** schema.sql**

>     schema.sql



> ** Spec.hs**

>     Spec.hs

