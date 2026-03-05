REP: XXX
Title: ROS Skill Definitions
Author: Séverin Lemaignan <severin.lemaignan@iiia.csic.es>
Status: Draft
Type: Standards Track
Content-Type: text/x-rst
Created: 05-Mar-2026
Post-History:


Abstract
========

This REP introduces the concept of *skills* as a standard mechanism
for declaring and documenting the high-level capabilities of a robot
within the ROS 2 ecosystem.  A skill is a curated ROS 2 action,
service, or topic that represents a useful, self-contained capability
(e.g., navigate to a pose, say a sentence, grasp an object).  Each
skill is described by a YAML manifest referred from (or directly embedded in)
the ``package.xml`` file of the ROS 2 package that defines it.  This manifest
specifies the skill's identifier, interface type, data type, parameters, and
functional domain.  A formal JSON Schema validates these manifests. Packages
that *implement* a skill declare this via an ``<implements>`` export tag,
enabling automatic discovery of available skill providers.  This REP
standardizes the manifest format, the validation schema, the domain taxonomy,
and the integration with the existing ``package.xml`` and ament build tooling,
thereby enabling interoperability across robots and facilitating both
high-level documentation of a robot's capabilities, as well as AI-based task
planning and decision-making.

Copyright
=========

This document has been placed in the public domain.

Specification
=============

This section defines the skill manifest format, the mechanism for
declaring skill implementations, the validation schema, the domain
taxonomy, and the integration with existing ROS 2 tooling.


High-Level Overview
--------------------

The skill framework introduces three distinct concepts that together
form a complete system for declaring, implementing, and invoking
robot capabilities:

1. **Skill definitions** (or *skill declarations*) are formal
   descriptions of a robot capability.  A skill definition specifies
   *what* the skill does, which ROS 2 interface it uses (action,
   service, or topic), its data type, its parameters, and the
   functional domain(s) it belongs to.  A skill definition is
   authored as a YAML manifest and embedded in the ``package.xml``
   of a dedicated *skill definition package* (e.g.,
   ``navigation_skills``).  Skill definition packages are organised
   by domain (see `Domain Taxonomy`_).

2. **Skill implementations** are regular ROS 2 nodes that realise
   one or more skill definitions.  A skill implementation declares
   which skills it provides via ``<implements>`` export tags in its
   own ``package.xml``.  This explicit declaration enables tooling
   to discover all available implementations of a given skill,
   supporting interoperability and substitutability across vendors.
   Skills may be *hierarchical*: a skill implementation may itself
   invoke other skills to carry out its work.  For example, a
   ``navigate`` skill might internally call a ``navigate_to_pose``
   skill, or a ``grasp`` skill might rely on
   ``execute_cartesian_trajectory``.

3. **Skill message types** define the ROS 2 interface data types
   (actions, services, or messages) used by each skill.  These
   types are defined alongside the skill definitions, within the
   same skill definition package.

   To enable runtime introspection and generic skill controllers and executors
   -- i.e., nodes that can invoke and monitor any skill without skill-specific
   code -- all skill interface types MUST include the following fields from the
   ``std_skills`` package [#std-skills]_:

   - ``std_skills/Meta meta`` -- present in the goal (for
     actions), the request (for services), and the message
     (for topics).  Contains metadata common to all skill
     invocations, such as priority and requester identity.

   - ``std_skills/Result result`` -- present in the result of
     actions and the response of services.  Provides a
     uniform way to report success, failure, and error codes.

   - ``std_skills/Feedback feedback`` -- present in the
     feedback of actions.  Provides uniform progress
     reporting.

   By mandating these common fields, any skill -- regardless of
   its domain or specific parameters -- can be invoked, monitored,
   and cancelled through a single, generic executor.


Skill Definition Manifest
--------------------------

A skill is defined by a manifest written in YAML or JSON,
associated with a ``<skill>`` element in the ``<export>`` section
of a ``package.xml`` file (format 3, as specified in
REP 149 [#rep149]_).

The manifest can be provided in two ways:

1. **Inline:** The manifest content is placed directly inside the
   ``<skill>`` element, with a ``content-type`` attribute set to
   ``"yaml"`` or ``"json"``:

   .. code-block:: xml

       <skill content-type="yaml">
         id: navigate_to_pose
         version: 1.0.0
         # ...
       </skill>

   .. note::

       If the inline YAML or JSON content contains XML reserved
       characters (such as ``&`` or ``<``),
       these characters MUST either be escaped using standard XML
       entities (e.g., ``&amp;``, ``&lt;``) or the entire content
       MUST be wrapped in a ``CDATA`` section:

       .. code-block:: xml

           <skill content-type="yaml"><![CDATA[
             id: my_skill
             description: |
               Uses <special> & "reserved" characters.
           ]]></skill>

2. **External file:** The manifest is stored in a separate file
   (e.g., ``navigate_to_pose-manifest.yml`` or
   ``navigate_to_pose-manifest.json``), and the ``<skill>``
   element references it via a ``src`` attribute:

   .. code-block:: xml

       <skill content-type="yaml" src="navigate_to_pose-manifest.yml" />

   The file path is relative to the directory containing the
   ``package.xml``.  The file format (YAML or JSON) is inferred
   from the file extension (``.yml``/``.yaml`` for YAML, ``.json``
   for JSON).

A single ``package.xml`` MAY contain multiple ``<skill>`` elements
(inline, external, or mixed), each defining a distinct skill.

The YAML manifest MUST contain the following fields:

``id``
    A unique, snake_case identifier for the skill (e.g.,
    ``navigate_to_pose``).  The identifier MUST be unique within
    the package.

``version``
    A semantic version string (``MAJOR.MINOR.PATCH``) following
    the conventions of `Semantic Versioning 2.0.0`_.

``description``
    A human-readable description of the skill's purpose and
    behaviour.

``interface``
    The ROS 2 communication pattern used by the skill.  MUST be
    one of: ``action``, ``service``, or ``topic``.

``datatype``
    The fully qualified ROS 2 interface type (e.g.,
    ``navigation_skills/action/NavigateToPose.action``).

The manifest MAY additionally contain:

``default_interface_path``
    The default ROS 2 name (topic/action/service path) under which
    the skill is expected to be exposed.  Implementations MAY remap
    this path.  The convention is ``/skill/<skill_id>``.

``functional_domains``
    A list of one or more domains to which the skill belongs.
    See `Domain Taxonomy`_ below.

``parameters``
    A mapping with an ``in`` key containing a list of input parameter
    descriptors.  Each parameter descriptor contains:

    ``name``
        The parameter name.  Dot-separated names (e.g.,
        ``meta.priority``) indicate sub-messages.

    ``type``
        The parameter type.  This can be a primitive type
        (``boolean``, ``integer``, ``float``, ``string``), or a
        reference to a ROS 2 message type using the notation
        ``package/msg/Type``.

    ``required``
        A boolean indicating whether the parameter is mandatory.
        Defaults to ``false`` if omitted.

    ``default``
        The default value, if any.

    ``description``
        A human-readable description of the parameter.

.. _Semantic Versioning 2.0.0: https://semver.org/


Example Skill Definition
''''''''''''''''''''''''

The following is a complete example of a skill definition embedded
in a ``package.xml``:

.. code-block:: xml

    <export>
      <build_type>ament_cmake</build_type>

      <skill content-type="yaml">
        id: navigate_to_pose
        version: 1.0.0
        description: |
          Make the robot autonomously navigate toward the
          desired pose.

        interface: action
        default_interface_path: /skill/navigate_to_pose
        datatype: navigation_skills/action/NavigateToPose.action
        functional_domains:
          - navigation
        parameters:
          in:
            - name: target_pose
              type: geometry_msgs/msg/PoseStamped
              required: true
              description: |
                The target pose to navigate to, in the
                specified reference frame.
            - name: safe_mode
              type: boolean
              default: true
              description: |
                If true, the robot navigates in safe mode
                (e.g., reduced speed near humans).
            - name: meta.priority
              type: integer
              default: 128
              description: |
                Between 0 and 255. Higher values mean this
                skill request will be prioritised.
      </skill>
    </export>


Declaring Skill Implementations
--------------------------------

A ROS 2 package that *implements* a skill defined in another package
declares this fact in the ``<export>`` section of its own
``package.xml`` using an ``<implements>`` tag:

.. code-block:: xml

    <export>
      <!-- ... -->
      <implements type="skill"
                  package="navigation_skills"
                  id="navigate_to_pose" />
    </export>

This enables tools to discover which packages provide implementations
for a given skill, similar to how ``pluginlib`` discovers plugin
providers.

The ``package`` attribute MUST reference the name of the ROS 2
package that contains the skill definition.  The ``id`` attribute
MUST match the ``id`` field of the corresponding skill manifest.


Validation Schema
-----------------

Skill manifests MUST be validated against a JSON Schema.  The
normative schema is maintained in the ``architecture_schemas``
package [#arch-schemas]_ and is available at:

    ``schemas/skill.schema.json``

The schema enforces the presence of required fields, validates
types, and checks enumerated values (e.g., the ``interface``
field).

An ament lint extension, ``ament_archlint`` [#arch-tools]_,
provides automated validation of skill manifests as part of the
standard ``ament_lint_auto`` pipeline.  Packages that define or
implement skills SHOULD include the following test dependency:

.. code-block:: xml

    <test_depend>ament_cmake_archlint</test_depend>

This allows skill manifest validation to be integrated into
continuous integration workflows and the ROS 2 build farm.


Domain Taxonomy
---------------

Skills are organised into *functional domains*.  A domain groups
related skills that address a common area of robot functionality.
The initial set of domains is:

===================  ================================================
Domain               Description
===================  ================================================
``communication``    Speech, dialogue, and language interaction
``interaction``      Non-verbal interaction (gaze, gestures, LEDs)
``manipulation``     Grasping, placing, and object handling
``management``       System and resource management
``motions``          Joint and Cartesian trajectory execution
``navigation``       Autonomous mobility and localisation
``reasoning``        Inference, planning, and decision-making
``UI``               User interface elements and displays
``external_bridge``  Bridging to external systems and services
``other``            Skills that do not fit into any other domain
===================  ================================================

Each domain corresponds to a dedicated ROS 2 package that contains
the skill definitions belonging to that domain (e.g.,
``communication_skills``, ``navigation_skills``).  This package
also provides the ROS 2 interface definitions (``.action``,
``.srv``, ``.msg`` files) required by its skills.

New domains MAY be proposed following the community process
described in `Proposing New Skills and Domains`_.


Initial Skill Catalogue
''''''''''''''''''''''''

The following is the initial set of standard skills proposed by
this REP, organised by domain.  The full, machine-readable
catalogue is available at [#skills-json]_.

**Communication:**

- ``say`` -- Speak out the provided input text.
- ``chat`` -- Start a dialogue with a defined purpose.
- ``ask`` -- A specialisation of ``chat`` for question-answering.
- ``ask_human_for_help`` -- Request assistance from a nearby human.

**Interaction:**

- ``look_at`` -- Define the gazing direction of the robot.
- ``look_for_human`` -- Search and localise specific humans.
- ``look_for_object`` -- Search and localise specific objects.
- ``set_expression`` -- Set the robot's facial expression or
  other expressive features.
- ``do_led_effect`` -- Perform light effects using the robot's
  LEDs.

**Manipulation:**

- ``grasp`` -- Command the robot to grasp a specified object.
- ``hand_grasp`` -- Control a robotic hand/gripper to open or
  close.
- ``place`` -- Command the robot to place an object at a
  specified location.

**Motions:**

- ``execute_joint_trajectory`` -- Execute a joint-space
  trajectory.
- ``execute_cartesian_trajectory`` -- Execute a Cartesian-space
  trajectory.
- ``replay_motion`` -- Execute a pre-recorded joint-space trajectory.

**Navigation:**

- ``navigate`` -- Autonomously navigate to a given location
  (abstract).
- ``navigate_to_pose`` -- Navigate toward a desired pose.
- ``navigate_to_waypoint`` -- Navigate toward a named waypoint.
- ``navigate_to_zone`` -- Navigate toward a named zone.

Relation to resource management and skill scheduling
''''''''''''''''''''''''''''''''''''''''''''''''''''

The skill framework -- and in particular the ``std_skills/Meta``
field included in every skill invocation -- is designed to carry
metadata that can support resource management and action
scheduling.  For instance, the ``meta.priority`` field allows a
skill executor to arbitrate between concurrent skill requests, and
future extensions to ``Meta`` could convey resource requirements,
deadlines, or pre-emption policies.

However, the design of such a resource manager or action scheduler
is out of scope for this REP.  This REP focuses
solely on defining the skill manifest format, the declaration and
discovery mechanism, and the common message structure.  How a robot
controller uses these declarations to allocate resources, resolve
conflicts, or sequence actions is left to higher-level frameworks
and is a potential subject for a future REP.

Proposing New Skills and Domains
''''''''''''''''''''''''''''''''

The skill catalogue is intended to grow over time through community
contributions.  The process for proposing additions is:

1. **New skill in an existing domain:**  Open an issue on the
   corresponding domain repository (e.g.,
   ``ros4hri/navigation_skills``) using the provided issue
   template.  The proposal is discussed publicly by the community.

2. **Amendment to an existing skill:**  Open an issue on the domain
   repository using the amendment template.  Amendments that change
   the interface type or the data type require a major version bump
   of the skill.

3. **New domain:**  Open a discussion on the ROS4HRI GitHub
   organisation [#ros4hri-discussions]_.  A new domain requires a
   corresponding ROS 2 package for its definitions.

Acceptance criteria for new skills include:

- The skill is novel and not a duplicate of an existing one.
- The skill would benefit a broad range of robots and applications.
- The skill is not tightly coupled to closed-source or proprietary
  technologies.
- The skill is well-documented.

Motivation
==========

Modern robots expose dozens or even hundreds of ROS 2 topics, services,
and actions.  While this fine-grained communication infrastructure is
powerful, it presents several challenges:

1. **Lack of a high-level "action API".**  There is no standard way to
   describe *what a robot can do* as opposed to *what interfaces it
   exposes*.  A navigation stack may expose multiple actions, services,
   and topics, but there is no machine-readable declaration that says
   "this robot can navigate to a pose".

2. **No standard documentation of a robot's public-facing API.** Robot
   integrators and end users need a clear, structured description of the
   high-level capabilities a robot offers -- its public "action API".  Today,
   this documentation is either absent, scattered across READMEs, or tightly
   coupled to implementation details. An introspectable list of implemented
   skills, alongside their skill manifests provides a single, authoritative
   source of truth for what a robot can do and how to invoke each capability.
   It also enable eg the automatic generation of documentation.

3. **Poor interoperability.**  Different robot vendors implement
   equivalent capabilities with different interface names, parameter
   conventions, and data types.  Without an agreed-upon semantic layer,
   every integrator must write bespoke glue code.

4. **AI-based planning and decision-making.**  Large Language Models
   (LLMs) and other AI planners increasingly drive robot behaviour.
   These planners need a well-defined, documented catalogue of what a
   robot can do -- its *skills* -- with formal parameter descriptions,
   to generate valid action sequences.

5. **Discoverability.**  There is no mechanism for a tool or planner to
   enumerate, at build time or run time, all the high-level capabilities
   a robot possesses.

The RobMoSys model [#robmosys]_ introduced a principled separation of
concerns between *skills* (what a robot can do), *services* (how those
skills are realised), and *components* (the software building blocks).
This REP brings a pragmatic, ROS-native implementation of the *skill*
layer to the broader ROS 2 community.

Rationale
=========

Design Decisions
----------------

**Why embed manifests in ``package.xml``?**  The ``package.xml`` file
(REP 149 [#rep149]_) is already the standard metadata file for every
ROS 2 package.  By reusing its ``<export>`` mechanism, skill
definitions require no new file format, no new parser, and no new
build system integration beyond what already exists.  The
``<export>`` section is explicitly designed for such extensions.

**Why a domain taxonomy?**  Organising skills into domains serves
two purposes: it provides a navigable structure for human users
browsing the skill catalogue, and it enables coarse-grained
filtering for AI planners that only need capabilities in a
specific area (e.g., "give me all navigation skills").

**Why separate definition from implementation?**  Decoupling the
*definition* of a skill (its interface contract) from its
*implementation* (the node that realises it) enables
interoperability.  Multiple vendors can provide competing
implementations of the same skill, and users can swap
implementations without changing their task-level code or
planner configuration.

**Why JSON Schema for validation?**  JSON Schema is a mature,
well-tooled standard for validating structured data.  The YAML
content of a skill manifest is trivially convertible to JSON for
validation purposes.  Reusing JSON Schema avoids inventing a
custom validation language.


Relationship to RobMoSys
-------------------------

The skill concept in this REP is strongly influenced by the
RobMoSys model [#robmosys]_, which defines a layered architecture
separating *tasks* (what the robot should achieve), *skills*
(parameterised capabilities), *services* (service-level
coordination), and *components* (software building blocks).

This REP focuses on the *skill* layer, providing a concrete,
ROS-native mechanism for defining and discovering skills.  It does
not prescribe how skills are composed into tasks, nor how they are
mapped to underlying components, leaving these concerns to
higher-level frameworks and planners.


Relationship to the OSRF Capabilities Package
----------------------------------------------

A closely related idea was explored in ROS 1 through the OSRF
``capabilities`` package [#osrf-capabilities]_, developed as part
of the Robotics-in-Concert (ROCON) project.  That package
introduced the concept of an *appable robot* whose high-level
capabilities could be declared, discovered, and managed at run
time via a capability server.

This REP shares the same fundamental motivation: providing a
standard way to describe *what a robot can do* rather than just
*what interfaces it exposes*.  However, the approach differs in
several important ways:

- **Build-time vs. run-time metadata.**  The OSRF capabilities
  package relied on a run-time capability server to manage the
  lifecycle of capabilities.  This REP takes a build-time-first
  approach, embedding skill metadata directly in ``package.xml``
  so that it is available for static analysis, build-farm
  validation, and documentation generation without requiring a
  running system.

- **ROS 2 ecosystem integration.**  The capabilities package was
  designed for ROS 1 (catkin) and was never ported to ROS 2.
  This REP is designed natively for ROS 2 and integrates with
  the ament build system and the REP 149 package manifest
  format.

- **Formal schema validation.**  Skill manifests in this REP are
  validated against a JSON Schema, enabling automated compliance
  checks in CI pipelines via ``ament_archlint``.

The OSRF capabilities package demonstrated the value of a
capability abstraction layer and can be considered an important
predecessor to this proposal.

Backwards Compatibility
========================

This REP introduces new functionality and does not modify any
existing ROS 2 interfaces, message types, or build system
behaviour.

- The ``<skill>`` export tag is a new, custom export element.
  Existing tools that parse ``package.xml`` will ignore unrecognised
  export tags, so no breakage is expected.

- The ``<implements>`` export tag follows the same principle.

- The ``ament_archlint`` validation tool is opt-in: packages must
  explicitly add it as a test dependency to enable validation.

- Existing packages that already use skill-like patterns (e.g.,
  move_base actions, MoveIt planning interfaces) are not affected.
  They can optionally adopt the skill framework by adding the
  appropriate manifest and ``<implements>`` tags.
  If the maintainers of these packages do not wish to adopt the skill
  interface directly, writing a thin adapter layer is easy.


Reference Implementation
=========================

A complete reference implementation is available and is being
prepared for release on the ROS 2 build farm.  It consists of the
following components:

1. **Skill definition packages:**  A set of ROS 2 packages
   providing skill definitions and their associated ROS 2
   interface types, organised by domain:

   - ``std_skills`` (common base types) [#std-skills]_
   - ``communication_skills`` [#comm-skills]_
   - ``interaction_skills`` [#int-skills]_
   - ``manipulation_skills`` [#manip-skills]_
   - ``motions_skills`` [#motion-skills]_
   - ``navigation_skills`` [#nav-skills]_

2. **Validation schema:**  The ``architecture_schemas`` package
   [#arch-schemas]_ containing the JSON Schema for skill manifests.

3. **Validation tooling:**  The ``architecture_tools`` package
   [#arch-tools]_ containing ``ament_archlint``, an ament lint
   extension that validates skill manifests against the schema.

4. **Documentation and catalogue:**  The full skill catalogue with
   human-readable documentation is available at
   [#skills-doc]_.

This work is co-maintained by PAL Robotics and the IIIA-CSIC
(Spanish National Institute for AI Research).


References
==========

.. [#robmosys] RobMoSys: Separation of Levels and Separation of
   Concerns
   (https://robmosys.eu/wiki/general_principles:separation_of_levels_and_separation_of_concerns)

.. [#rep149] REP 149, Package Manifest Format Three Specification
   (https://ros.org/reps/rep-0149.html)

.. [#skills-doc] ROS4HRI Standard Skills Documentation
   (https://ros4hri.github.io/skills/)

.. [#skills-json] ROS4HRI Skills Catalogue (JSON)
   (https://ros4hri.github.io/skills.json)

.. [#arch-schemas] architecture_schemas -- JSON Schema for skill
   manifests
   (https://github.com/ros4hri/architecture_schemas)

.. [#arch-tools] architecture_tools -- ament lint extension for
   skill validation
   (https://github.com/ros4hri/architecture_tools)

.. [#comm-skills] communication_skills
   (https://github.com/ros4hri/communication_skills)

.. [#int-skills] interaction_skills
   (https://github.com/ros4hri/interaction_skills)

.. [#manip-skills] manipulation_skills
   (https://github.com/ros4hri/manipulation_skills)

.. [#motion-skills] motions_skills
   (https://github.com/ros4hri/motions_skills)

.. [#nav-skills] navigation_skills
   (https://github.com/ros4hri/navigation_skills)

.. [#std-skills] std_skills
   (https://github.com/ros4hri/std_skills)

.. [#ros4hri-discussions] ROS4HRI Community Discussions
   (https://github.com/orgs/ros4hri/discussions)

.. [#osrf-capabilities] OSRF Capabilities -- ROS 1 capability
   declaration and management
   (https://github.com/osrf/capabilities)


..
   Local Variables:
   mode: indented-text
   indent-tabs-mode: nil
   sentence-end-double-space: t
   fill-column: 70
   coding: utf-8
   End:
