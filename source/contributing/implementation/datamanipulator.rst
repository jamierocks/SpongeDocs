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

The two methods ``copy()`` and ``asImmutable()`` are easy to implement. For both you just need to return a mutable or an immutable data manipulator respectively, containing the same data as the current instance.

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

.. tip::

    It may be beneficial to declare the fields of an ``ImmutableDataManipulator`` as ``final`` in order to
    prevent accidental changes.

3. Register the Key in the KeyRegistry
======================================

The next step is to register your ``KEYS`` in the ``KeyRegistry``:

.. code-block:: java

 public static void registerKeys() {
      keyMap.put("is_flying", makeSingleKey(Boolean.class, Value.class, of("IsFlying")));
 }


4. Implement the DataProcessor
==============================

Next up is the ``DataProcessor``. A ``DataProcessor`` implements your ``DataManipulator`` to work semi-natively with
Minecraft's objects. Unfortunately, since it's always easier to just directly implement all of the processing of a
particular set of data in one place, we have ``DataProcessor``\s to do just that. To simplify the implementation of
``DataProcessor``\s, ``AbstractDataProcessor`` was developed to reduce boilerplate code. It brought up two additional
``DataProcessor``\s, ``AbstractEntitySingleDataProcessor`` and ``AbstractEntityDataProcessor`` which are specifically
targeted at ``Entities`` based on ``net.minecraft.entity.entity``. These two ``processor``\s reduce the needed code and
leave us with this:


.. code-block:: java

 protected abstract M createManipulator();

  protected boolean supports(E entity) {
      return true;
  }

  protected abstract boolean set(E entity, T value);

  protected abstract Optional<T> getVal(E entity);

  protected abstract ImmutableValue<T> constructImmutableValue(T value);


createManipulator() method
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: java

 @Override
 protected HealthData createManipulator() {
    return new SpongeHealthData(20, 20);
 }

doesDataExist() method
~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: java

 @Override
 protected boolean doesDataExist(EntityLivingBase entity) {
    return true;
 }

set() method
~~~~~~~~~~~~

.. code-block:: java

 @Override
 protected boolean set(EntityLivingBase entity, Map<Key<?>, Object> keyValues) {
    entity.getEntityAttribute(SharedMonsterAttributes.maxHealth).setBaseValue(((Double) keyValues.get(Keys.MAX_HEALTH)).floatValue());
    entity.setHealth(((Double) keyValues.get(Keys.HEALTH)).floatValue());
    return true;
 }

getValues(DataContainer)
~~~~~~~~~~~~~~~~~~~~~~~~

.. code-block:: java

 @Override
 protected Map<Key<?>, ?> getValues(EntityLivingBase entity) {
    final double health = entity.getHealth();
    final double maxHealth = entity.getMaxHealth();
    return ImmutableMap.<Key<?>, Object>of(Keys.HEALTH, health, Keys.MAX_HEALTH, maxHealth);
 }


fill(DataContainer)
~~~~~~~~~~~~~~~~~~~

.. code-block:: java

 @Override
 public Optional<HealthData> fill(DataContainer container, HealthData healthData) {
    healthData.set(Keys.MAX_HEALTH, getData(container, Keys.MAX_HEALTH));
    healthData.set(Keys.HEALTH, getData(container, Keys.HEALTH));
    return Optional.of(healthData);
 }

remove(DataHolder)
~~~~~~~~~~~~~~~~~~

.. code-block:: java

 @Override
 public DataTransactionResult remove(DataHolder dataHolder) {
    return DataTransactionBuilder.builder().result(DataTransactionResult.Type.FAILURE).build();
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
