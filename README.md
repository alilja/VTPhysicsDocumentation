Dynamic Physics Engine Documentation
====================================

The system is designed to make upgrading as simple as possible. Just use ``loadModel`` like you normally would, except don't use ``addChild`` afterwards.

# System Structure

The key files are ``PhysicsConfigurator`` and ``RealWorldPhysics``. ``ArticulationVistor`` no longer exists &mdash; models are added on-the-fly using ``PhysicsConfigurator::addModel``. ``Scene.h`` contains a pointer to a scene's specific ``PhysicsConfigurator``, which allows you to access phyiscs-related functions like applying impulses, forces, and identifying specific nodes.

## PhysicsConfigurator

``Scene.h`` contains a convenience ``loadModel`` function that simply calls ``physics_config_->addModel``. ``addModel`` is the canonical way to add new elements to a scene, but for convenient upgrading, ``loadModel`` has been created.

``PhysicsConfigurator::addModel`` does two main things: it moves all models specified in the configuration file to the right location, and then sets up phyiscs to those that require it. It does *not* place the objects into the world &mdash; that is handled by ``RealWorldPhysics`` during its activation. ``addModel`` reads the configuration file for the model, scale, rotation, and location information.

The model is passed to a ``PhysicsVisitor``, which reads the model to identify which parts of need physics applied to them and the collision shape proxytypes. These are placed in ``PhysicsVisitor::node_collision_shapes_``, which can be accessed with ``PhysicsVisitor::get_collision_shapes()``.

``addModel`` next creates a "master" transformation node that it sticks everything under. This node is given the name specified by the ``model_id`` argument.

### Non-Colliding Objects

A ``static_matrix_transform`` is created containing the transformation of the non-colliding objects &mdash; that is, objects that have no relationship to the physical world. You might use this for things like a skybox, ground plants, or any other decorative object. All of the non-colliding objects are added to this matrix transform, which is then added to the ``master_node``. After this, these objects are done &mdash; ``RealWorldPhysics`` doesn't act on them.

### Colliding Objects

There are two types of colliding objects: dynamic objects, which can move, and static objects, which cannot. Unless explicitly specified in the configuration file by adding ``<physics>`` and ``<mass>`` tags, all objects are static. Because each of these objects needs to be handled separately within the same model group, each receives its own transformation node separate from the ``master_node.`` This new node is added to the ``master_node.``

OsgBullet's ``CreationRecord`` class is then used to create the rigid body. The scene graph and Bullet are tied together with the ``cr->_sceneGraph = matrix_transform`` line &mdash; Bullet doesn't care about the ``Geode``, only the ``MatrixTransform`` attached to it. The ``_shapeType`` is collected from the ``PhysicsVisitor``, and mass and restitution are applied. Then the ``btRigidBody*`` is created.

The last step is to create the ``Geode``. We need to attach a ``MotionState`` to it (I don't know why. I think this is the interface to Bullet, so if we move something around here Bullet learns about it), so we get the one from the ``btRigidBody`` we just created. Then we apply the translation steps to the motionstate (which I believe updates Bullet), and applies the transforms to the world. Finally, we stick this shiny new rigid body in ``dynamic_bodies_``.

## RealWorldPhysics

Scenes are created on the fly, so we can't add the rigid bodies directly to the dynamics world from ``PhyiscsConfigurator`` because it doesn't exist yet. It doesn't actually get created until the activation method of ``RealWorldPhysics``, so that's where we finally add the rigid bodies we made to the world. We iterate over ``dynamic_bodies_`` and add each of them to the world using ``addRigidBody``.

You'll notice the ``COL_*`` elements &mdash; these are collision filters. Because the player object is a ghost object, it can't interact with physics objects. This manifested as a bug whereby all the physics objects could interact with each other, but they'd stop the player dead in their tracks, regardless of mass. To fix this, all physical objects are given collision filters (``COL_*``), including the player object.

# Methods

Any time you want to interact with the physics of objects, you do it through ``PhysicsConfigurator`` (which is also where you should add new physics-related methods). From a scene, you can get access to these methods from ``physics_config_->*``.

* ``addModel``: Adds a model to the scene. See also ``loadModel``, which does the exact same thing but uses the old-style formatting.

* ``find_rigid_body``: Given a name, will return a ``btRigidBody*``. Don't use this unless you *really* have to.

* ``apply_impulse``: Applies an impulse (actually a Bullet force, for reasons of Bullet's bizarre legacy support) to an object.

* ``apply_force``: Applies a constant force to an object (again, for legacy reasons, this is actually considered an additional force of *gravity*. Don't complain to me, man.).

* ``apply_torque``: Applies a torque impulse to an object.


# Configuration Files

## Header

Configuration files for the new physics engine look generally the same. They start with the usual header:

    <?xml version="1.0"?>
    <!DOCTYPE scene SYSTEM "Scene.dtd">

And then the ``scene`` tag:

    <scene name="Maze">

This tells VirtuTrace what your scene's name is (which you plug in to ``SceneManager.xml``) and gets everything ready for your models.

## Models

All models are contained within the ``<models>`` tag:

    <?xml version="1.0"?>
    <!DOCTYPE scene SYSTEM "Scene.dtd">

    <scene name="Maze">
        <models>
            <model id="maze"></model>
        </models>
    </scene>

A model has a few parameters you can add:

* ``file``: specify the model's file location, e.g. ``<file>${VT_MODELS_PATH}/Maze/maze.obj</file>``

* ``position``: ``<position x="0" y="-100" z="200" />`` You can also specify the name of another node, and that node's position will be used for this object's starting position. If you want this object to *stay* with that node, you'll have to do that in code.

* ``rotation``: ``<rotation x="90" y="0" z="0" />``

There's also two sub-groups: ``physics`` and ``collidables``. You need both subgroups if you want physics applied to your objects. An object *without* physics will not interact with the world in any way.

The ``collidables`` subgroup allows you to specify regular expressions and collision shapes for parts of the model you want to have collisions. This allows multi-part models to have some areas with collisions and others without.

    <collidables>
        <geode shape="BOX">floor\.[0-9]+_Plane.*</geode>
        <geode shape="BOX">Cube\.[0-9]+_Cube\.[0-9]+</geode>
    </collidables>

Then, you need to specify the ``physics`` properties you'd like to apply. Note that these are applied to the *entire* model, and different parts cannot have different physical properties. Presently, you can specificy ``mass`` and ``restitution``.

    <physics>
        <mass>10.0</mass>
        <restitution>0.8</restitution>
    </physics>

* ``mass``: The mass of the object. Defaults to 0. If set to 0, the object is *static* and will interact with colliding objects but cannot be moved itself. Useful for floors, walls, etc.

* ``restitution``: How quickly movement decays. Ranges from 0.0 &ndash; 1.0, but high values create hilarity.


