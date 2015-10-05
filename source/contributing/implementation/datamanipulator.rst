=============================
Implementing DataManipulators
=============================

This is a guide for contributors who want to help with Data API implementation by creating DataManipulators.
An updated list of DataManipulators to be implemented can be found at
`SpongeCommon Issue #8 <https://github.com/SpongePowered/SpongeCommon/issues/8>`_.

To fully implement a ``DataManipulator`` these steps must be followed:

1. Implement the ``DataManipulator`` itself
#. Implement the ``ImmutableDataManipulator``

When these steps are complete, the following must also be done:

3. Register the ``Key`` in the ``KeyRegistry``
#. Implement the ``DataProcessor``
#. Implement the ``DataBuilder``
#. Implement the ``ValueProcessor`` for each value being represented by the ``DataManipulator``
#. Register everything in ``SpongeGameRegistry``

.. note::
    Make sure you follow our :doc:`../guidelines`.

1. Implementing the DataManipulator
===================================

The naming convention for ``DataManipulator`` implementations is the name of the interface prefixed with "Sponge".
So to implement the ``HealthData`` interface, we create a class named ``SpongeHealthData`` in the appropriate package.
For implementing the ``DataManipulator`` first have it extend an appropriate abstract class from the
``org.spongepowered.common.data.manipulator.mutable.common`` package. The most generic there is ``AbstractData``
but there are also abstractions that reduce boilerplate code even more for some special cases like
``DataManipulator``\ s only containing a single value.

.. code-block:: java

    public class SpongeHealthData extends AbstractData<HealthData, ImmutableHealthData> implements HealthData {
        [...]
    }

There are two type arguments to the AbstractData class. The first is the interface implemented by this class, the
second is the interface implemented by the corresponding ``ImmutableDataManipulator``.

The Constructor
~~~~~~~~~~~~~~~

In most cases while implementing an abstract Manipulator you want to have two constructors:

* One without arguments (no-args) which calls the second
* and one single-argument constructor which is actually used for the provided values

The second constructor must

* make a call to the ``AbstractData`` constructor, passing the class reference for the implemented interface.
* call the ``registerGettersAndSetters()`` method

.. code-block:: java

    public SpongeHealthData() {
        this(20D, 20D);
    }

    public SpongeHealthData(double currentHealth, double maxHealth) {
        super(HealthData.class);
        this.currentHealth = currentHealth;
        this.maximumHealth = maxHealth;
        this.registerGettersAndSetters();
    }

Accessors defined by the Interface
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The interface we implement specifies some methods to access ``Value`` objects. For ``HealthData``, those are
``health()`` and ``maxHealth()``. Every call to those should yield a new ``Value``.

.. code-block:: java

    public MutableBoundedValue<Double> health() {
        return new SpongeBoundedValue<>(Keys.HEALTH, this.maximumHealth,
            ComparatorUtil.doubleComparator(), 0D, (double) Float.MAX_VALUE,
            this.currentHealth);
    }

Since we use a bounded value, our constructor is slightly longer. Therefore, the arguments are spread over multiple
lines for this example. First the ``Key`` to properly identify the value, then the *default* value. Afterwards,
on the second line, three arguments follow that are specific to bounded values, as they provide a comparator to
use and minimum and maximum values. The last argument specifies the current value. If it is not present, the
default value will be used instead.

Copying and Serialization
~~~~~~~~~~~~~~~~~~~~~~~~~

The two methods ``copy()`` and ``asImmutable()`` are not much work to implement. For both you just need to return
a mutable or an immutable data manipulator respectively, containing the same data as the current instance.

The method ``toContainer()`` is used for serialization purposes. Use a ``MemoryDataContainer`` as the result
and apply to it the values stored within this instance. A ``DataContainer`` is basically a map mapping ``DataQuery``\ s
to values. Since a ``Key`` always contains a corresponding ``DataQuery``, just use those using the convenience methods.

.. code-block:: java

    public DataContainer toContainer() {
        return new MemoryDataContainer()
            .set(Keys.HEALTH, this.currentHealth)
            .set(Keys.MAX_HEALTH, this.maximumHealth);
    }

registerGettersAndSetters()
~~~~~~~~~~~~~~~~~~~~~~~~~~~

A ``DataManipulator`` also provides methods to get and set data using keys. The implementation for this is handled
by ``AbstractData``, but we must tell it which data it can access and how. Therefore, in the
``registerGettersAndSetters()`` method we need to do the following for each value:

* register a ``GetterFunction<Object>`` to directly get the value
* register a ``SetterFunction<Object>`` to directly set the value
* register a ``GetterFunction<Value<?>>`` to get the mutable ``Value``

Those ``GetterFunction`` and ``SetterFunction`` objects only contain one method, so Java 8 Lambdas can be used.

.. code-block:: java

 private void registerGettersAndSetters() {
      registerFieldGetter(Keys.HEALTH, () -> SpongeHealthData.this.currentHealth);
      registerFieldSetter(Keys.HEALTH, value -> SpongeHealthData.this.currentHealth ((Number) value).doubleValue()));
      registerKeyValue(Keys.HEALTH, SpongeHealthData.this::health);
  }

That's it. The ``DataManipulator`` should be done now.

2. ImmutableDataManipulator
===========================

Implementing the ``ImmutableDataManipulator`` is similar to implementing the mutable one.

The only differences are:

* The class name is formed by prefixing the mutable ``DataManipulator``\ s name with ``Immutable``
* Inherit from ``ImmutableAbstractData`` instead
* Instead of ``registerGettersAndSetters()``, the method is called ``registerGetters()``

When creating ``ImmutableDataHolder``\ s or ``ImmutableValue``\ s, check if it makes sense to use the
``ImmutableDataCachingUtil``. For example if you have ``WetData`` which contains nothing more than a boolean, it
is more feasible to retain only two cached instances of ``ImmutableWetData`` - one for each possible value. For
manipulators and values with many possible values (like ``SignData``) however, caching may prove too expensive.

.. tip::

    It may be beneficial to declare the fields of an ``ImmutableDataManipulator`` as ``final`` in order to
    prevent accidental changes.

3. Register the Key in the KeyRegistry
======================================

The next step is to register your ``Key``\ s to the ``KeyRegistry``. To do so, locate the
``org.spongepowered.common.data.key.KeyRegistry`` class and find the static ``registerKeys()`` function.
There add a line to register (and create) your used keys.

.. code-block:: java

    public static void registerKeys() {
        keyMap.put("max_health", makeSingleKey(Double.TYPE, MutableBoundedValue.class, of("MaxHealth")));
    }

The ``keyMap`` maps strings to ``Key``\ s. The string used should be the corresponding constant name from
the ``Keys`` utility class in lowercase. The ``Key`` itself is created by one of the static methods
provided by ``KeyFactory``, in most cases ``makeSingleKey``. ``makeSingleKey`` requires first a class reference
for the underlying data, which in our case is a "Double", then a class reference for the ``Value`` type used.
The third argument is the ``DataQuery`` used for serialization. It is created from the statically imported
``DataQuery.of()`` method accepting a string. This string should also be the constant name, stripped of
underscores and capitalization changed to upper camel case.

.. tip::
    For primitive types (like double, int, boolean), use the constant ``TYPE`` provided in its wrapper class, not the class reference.

4. Implement the DataProcessor
==============================

Next up is the ``DataProcessor``. A ``DataProcessor`` serves as a bridge between our ``DataManipulator`` and
Minecraft's objects. Whenever any data is requested from or offered to ``DataHolders`` that exist in Vanilla
Minecraft, those calls end up being handled by a ``DataProcessor`` or a ``ValueProcessor``.

In order to reduce boilerplate code, the ``DataProcessor`` should inherit from the appropriate abstract class in
the ``org.spongepowered.common.data.processor.common`` package. Since health can only be present on certain
entities, we can make use of the ``AbstractEntityDataProcessor`` which is specifically
targeted at ``Entities`` based on ``net.minecraft.entity.entity``.

.. code-block:: java

    public class HealthDataProcessor extends AbstractEntityDataProcessor<EntityLivingBase, HealthData, ImmutableHealthData> {
        public HealthDataProcessor() {
            super(EntityLivingBase.class);
        }
        [...]
    }

Depending on which abstraction you use, the methods you have to implement may differ greatly, depending on how
much implementation work already could be done in the abstract class. Generally, the methods can be categorized.

Validation Methods
~~~~~~~~~~~~~~~~~~

Always return a boolean value. If the method is called ``supports()`` it should perform a general check if the supplied target generally supports the kind of data handled by our ``DataProcessor``.

For our ``HealthDataProcessor`` ``supports()`` is implemented by the ``AbstractEntityDataProcessor``. Per
default, it will return true if the supplied argument is an instance of the class specified when calling the
``super()`` constructor.

Instead, we are required to provide a ``doesDataExist()`` method. Since the abstraction does not know how to
obtain the data, it leaves this function to be implemented. As the name says, the method should check if the data
already exists on the supported target. For the ``HealthDataProcessor``, this always returns true, since every
living entity always has health.

.. code-block:: java

    protected boolean doesDataExist(EntityLivingBase entity) {
        return true;
    }

Setter Methods
~~~~~~~~~~~~~~

A setter method receives a ``DataHolder`` of some sort and some data that should be applied to it, if possible.

The ``DataProcessor`` interface defines a ``set()`` method accepting a ``DataHolder`` and a ``DataManipulator`` which returns a ``DataTransactionResult``. Depending on the abstraction class used, some of the necessary functionality might already be implemented.

In this case, the ``AbstractEntityDataProcessor`` takes care of most of it and just requires a method to set some values to return ``true`` if it was successful and ``false`` if it was not. Creation of the ``DataTransactionResult`` is fully handled by the abstract class.

.. code-block:: java

    protected boolean set(EntityLivingBase entity, Map<Key<?>, Object> keyValues) {
        entity.getEntityAttribute(SharedMonsterAttributes.maxHealth)
            .setBaseValue(((Double) keyValues.get(Keys.MAX_HEALTH)).floatValue());
        entity.setHealth(((Double) keyValues.get(Keys.HEALTH)).floatValue());
        return true;
    }

.. tip::

    To understand ``DataTransactionResult`` \ s, check the :doc:`corresponding docs page
    <../../plugin/data/transactions>` and refer to the `DataTransactionBuilder API
    Docs <https://jd.spongepowered.org/index.html?org/spongepowered/api/data/DataTransactionBuilder.html>`_ to
    create one.

Removal Methods
~~~~~~~~~~~~~~~

The ``remove()`` method attempts to remove data from the ``DataHolder`` and returns a ``DataTransactionResult``.

Since it is impossible for an ``EntityLivingBase`` to not have any health, this operation will always fail on the ``HealthDataProcessor``.

.. code-block:: java

    public DataTransactionResult remove(DataHolder dataHolder) {
        return DataTransactionBuilder.failNoData();
    }


Getter Methods
~~~~~~~~~~~~~~

Getter methods obtain data from a ``DataHolder`` and return an optional ``DataManipulator``. The
``DataProcessor`` interface specifies the methods ``from()`` and ``createFrom()``, the difference being that
``from()`` will return ``Optional.empty()`` if the data holder is compatible, but currently does not contain the
data, while ``createFrom()`` will provide a ``DataManipulator`` holding default values in that case.

Again, ``AbstractEntityDataProcessor`` will provide most of the implementation for this and only requires a
method to get the actual values present on the ``DataHolder``. This method is only called after ``supports()``
and ``doesDataExist()`` both returned true.

.. code-block:: java

    protected Map<Key<?>, ?> getValues(EntityLivingBase entity) {
        final double health = entity.getHealth();
        final double maxHealth = entity.getMaxHealth();
        return ImmutableMap.<Key<?>, Object>of(Keys.HEALTH, health, Keys.MAX_HEALTH, maxHealth);
    }

Filler Methods
~~~~~~~~~~~~~~

A filler method is different from a getter method in that it receives a ``DataManipulator`` to fill with values. These values either come from a ``DataHolder`` or have to be deserialized from a ``DataContainer``. The method returns ``Optional.empty()`` if the ``DataHolder`` is incompatible.

``AbstractEntityDataProcessor`` already handles filling from ``DataHolders`` by creating a ``DataManipulator`` from the holder and then merging it with the supplied manipulator, but the ``DataContainer`` deserialization it can not provide.

.. code-block:: java

    public Optional<HealthData> fill(DataContainer container, HealthData healthData) {
        healthData.set(Keys.MAX_HEALTH, DataUtil.getData(container, Keys.MAX_HEALTH));
        healthData.set(Keys.HEALTH, DataUtil.getData(container, Keys.HEALTH));
        return Optional.of(healthData);
    }

Other Methods
~~~~~~~~~~~~~

Depending on the abstract superclass used, some other methods may be required. For instance,
``AbstractEntityDataProcessor`` needs to create ``DataManipulator`` instances in various points. It can't do this
since it knows neither the implementation class nor the constructor to use. Therefore it utilizes an abstract
function that has to be provided by the final implementation. This does nothing more than create a
``DataManipulator`` with default data.

If you implemented your ``DataManipulator`` as recommended, you can just use the empty constructor.

.. code-block:: java

    protected HealthData createManipulator() {
        return new SpongeHealthData();
    }


5. Implement the DataBuilder
============================

6. Implement the ValueProcessor
===============================

7. Register everything in SpongeGameRegistry
============================================

When finally done, you have to register everything in ``SpongeGameRegistry``. As always, add all necessary imports, then the ``DataProcessor`` and
``DataBuilder``:

.. code-block:: java

 private void setupSerialization() {
  final FlyingDataProcessor flyingDataProcessor = new FlyingDataProcessor();
        final FlyingDataBuilder flyingDataBuilder = new FlyingDataBuilder();
        service.registerBuilder(FlyingData.class, flyingDataBuilder);
         dataRegistry.registerDataProcessorAndImpl(FlyingData.class, SpongeFlyingData.class, ImmutableFlyingData.class, ImmutableSpongeFlyingData.class, flyingDataProcessor, flyingDataBuilder);
 }

And finally the ``ValueProcessor``:

.. code-block:: java

 private void setupSerialization() {
  dataRegistry.registerValueProcessor(Keys.IS_FLYING, new IsFlyingValueProcessor());
 }

Examples
========

Several ``DataManipulators`` have already been implemented. Have a look at them to get a quick overview on how to implement your own ``Manipulator``.
Here are some examples which might help you understanding the concept of ``DataManipulator``:

* `VelocityData <https://github.com/SpongePowered/SpongeCommon/commit/ab47f2681dd382a44f1d32d92858bd29c2910ff3>`_
* `BreathingData <https://github.com/SpongePowered/SpongeCommon/commit/f461697e0a6de7840e7cdb6e739d97cb176d7617>`_
* `FoodData <https://github.com/SpongePowered/SpongeCommon/commit/19c13cb71ea3e1d8cd67372b7f272fe298c21902>`_
