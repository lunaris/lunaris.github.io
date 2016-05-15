---
layout: post
title: MonadTransControl
---
Haskell's support for first-class actions permits the elegant implementation of
many concepts. The
[postgresql-simple](https://hackage.haskell.org/package/postgresql-simple "postgresql-simple")
package, for instance, provides the following combinator
(users of the library will recognise that this is an exact
type, but it is morally equivalent for the purposes of this post):

{% highlight haskell %}
withTransaction :: Connection -> IO () -> IO ()
{% endhighlight %}

Given some action `act` which makes executes a series of PostgreSQL queries
`q1`, `q2` and so on, the expression `withTransaction m` will execute:

{% highlight haskell %}
BEGIN;
  q1
  q2
  ...
COMMIT;
{% endhighlight %}

This is far neater and more robust than an alternative pair of combinators,
`begin` and `commit` which must be called (and potentially nested) correctly by
users wishing to execute transactions.

Of course, there is trouble in paradise. `withTransaction`'s use of the `IO`
type means that (in theory) it could do far more than execute the SQL above,
such as wiping the hard disk or launching the proverbial missiles. Better would
be to ascribe `withTransaction` a type that describes precisely the side effects
it depends upon and exhibits. We can do this by capturing the interface
provided by `withTransaction` and its database-handling companions in a type
class:

{% highlight haskell %}
class Monad m => MonadPostgreSQL m where
  ...
  withTransactionM :: m a -> m a
{% endhighlight %}

While the `IO`-backed behaviour can then be recovered with a dedicated
instance:

{% highlight haskell %}
instance MonadPostgreSQL IO where
  withTransactionM
    = withTransaction
{% endhighlight %}

it is common to provide a _monad transformer_ which can wrap an existing type
so that it is capable of providing an instance.  This is a pattern captured in
the popular [mtl](https://hackage.haskell.org/package/mtl "mtl") and
[transformers](https://hackage.haskell.org/package/transformers "transformers")
packages. The notion of a read-only global environment, for example, is
captured by the `MonadReader` type class and accompanying `ReaderT` monad
transformer:

{% highlight haskell %}
class Monad m => MonadReader r m where
  ask :: m r

newtype ReaderT r m a
  = ReaderT { runReaderT :: r -> m a }

instance Monad m => MonadReader r (ReaderT r m) where
  ask
    = ReaderT (\env -> pure env)
{% endhighlight %}

Where `r` is the type of the environment being made accesible to computations
in the monad through the `ask` function. An oft-cited gripe with the `mtl` way
of doing things is that, while the instance above is "the only one that
matters" since it describes how the `ReaderT` transformer can make an existing
monad compatible with the `MonadReader` class, one must also define
"pass-through" instances for _any other transformer_ that is capable of
preserving this behaviour. For instance, if some monad `m` already implements
`MonadReader r`, the result of applying the identity transformer, `IdentityT`,
surely does too:

{% highlight haskell %}
newtype IdentityT m a
  = IdentityT { runIdentityT :: m a }

instance MonadReader r m => MonadReader r (IdentityT m) where
  ask
    = IdentityT ask
{% endhighlight %}

Such instances, while tedious, are at least typically simple to write. That is,
until we consider functions like `withTransactionM`. Suppose we define a
transformer, `PostgreSQLT`, so that we might bolt on PostgreSQL support to an
existing monad:

{% highlight haskell %}
newtype PostgreSQLT m a
  = PostgreSQLT { runPostgreSQLT :: m a }

instance Monad m => MonadPostgreSQL (PostgreSQLT m) where
  withTransactionM (PostgreSQLT act)
    = PostgreSQLT _
{% endhighlight %}

How might we define `withTransactionM`? We know that we want to use
`withTransaction` to wrap the supplied action (`act`):

{% highlight haskell %}
withTransactionM
  :: Monad m
  => PostgreSQLT m a
  -> PostgreSQLT m a

withTransactionM (PostgreSQLT act)
  = PostgreSQLT $ _ioToM $ withTransaction $ _mToIO act
{% endhighlight %}

There are two issues with this, illustrated by the holes `_ioToM` and `_mToIO`.
As the names may suggest, the issues are ones of moving between `IO`, the type
understood by `withTransaction`, and `m`, the type of the abstract monad we are
working in. The first hole, `_ioToM`, of type `IO a -> m a`, can be filled with
`liftIO` from the `MonadIO` type class, which provides an escape hatch for
monads based on `IO`:

{% highlight haskell %}
class Monad m => MonadIO m where
  liftIO :: IO a -> m a

withTransactionM
  :: MonadIO m
  => PostgreSQLT m a
  -> PostgreSQLT m a

withTransactionM (PostgreSQLT act)
  = PostgreSQL $ liftIO $ withTransaction $ _mToIO act
{% endhighlight %}

If one is familiar with monad transformers in general, it's relatively easy to
put together the instances for `MonadIO`. `IO` provides a natural base:

{% highlight haskell %}
instance MonadIO IO where
  liftIO
    = id
{% endhighlight %}

And the remaining "pass-through" instances simply hand the given `IO` action
downward until it hits the `IO` layer. For instance, in the cases of
`IdentityT` and `ReaderT`:

{% highlight haskell %}
instance MonadIO m => MonadIO (IdentityT m) where
  liftIO ioAct
    = IdentityT (liftIO ioAct)

instance MonadIO m => MonadIO (ReaderT r m) where
  liftIO ioAct
    = ReaderT (\_ -> liftIO ioAct)
{% endhighlight %}

Returning to our problem, we might imagine that the second hole, `_mToIO`, of
type `m a -> IO a` can be solved with a type class too, perhaps of the form:

{% highlight haskell %}
class Monad m => MonadBaseIO m where
  runInIO :: m a -> IO a

withTransactionM
  :: (MonadIO m, MonadBaseIO m)
  => PostgreSQLT m a
  -> PostgreSQLT m a

withTransactionM (PostgreSQLT act)
  = PostgreSQL $ liftIO $ withTransaction $ runInIO act
{% endhighlight %}

It turns out that `MonadBaseIO` is an instance of a more general pattern
captured by the `MonadTransControl`, `MonadBase` and `MonadBaseControl` classes
of the
[transformers-base](https://hackage.haskell.org/package/transformers-base "transformers-base") and
[monad-control](https://hackage.haskell.org/package/monad-control "monad-control") packages.
Unfortunately, working out the definitions of any of these classes is a bit
more involved than the casual stroll that saw us arrive at `MonadIO`. In the
remainder of this post we'll work towards this sort of behaviour and see how it
can be generalised to work with arbitrary stacks of monad transformers.
