.. _devicetree-intro:

Introduction to devicetree
##########################

This page provides an introduction to :ref:`devicetree` and how it is used in
Zephyr.

The following figure shows how devicetree is used by :ref:`Zephyr's build
system <build_overview>`:

.. figure:: zephyr_dt_build_flow.png
   :figclass: align-center

   Devicetree build flow

The build system generates a C header file which contains preprocessor
:ref:`macros with devicetree data <dt-macros>`. These macros can be referenced
by :ref:`device drivers <device_model_api>` and other C code by including
:file:`<devicetree.h>`. All macro identifiers that are directly generated by
the devicetree scripts start with ``DT_``.

Sometimes, information from devicetree is available using ``CONFIG_`` macros
generated from :ref:`Kconfig <kconfig>`. This happens when devicetree-related
information is referenced from Kconfig symbol definitions via :ref:`Kconfig
functions <kconfig-functions>`. See :ref:`dt_vs_kconfig` for some additional
comparisons with Kconfig.

This differs significantly from how devicetree is used on Linux. The
Linux kernel would instead read the entire devicetree data structure in its
binary form, parsing it at runtime in order to load and initialize device
drivers. Zephyr does not work this way because the size of the devicetree
binary and associated handling code would be too large to fit comfortably on
the relatively constrained devices Zephyr supports.

.. _dt-syntax:

Syntax and structure
********************

As the name indicates, a devicetree is a tree. The human-readable text format
for this tree is called DTS (for devicetree source), and is defined in the
`Devicetree specification`_.

.. _Devicetree specification: https://www.devicetree.org/

Here is an example DTS file:

.. code-block:: DTS

   /dts-v1/;

   / {
           a-node {
                   subnode_label: a-sub-node {
                           foo = <3>;
                   };
           };
   };

The ``/dts-v1/;`` line means the file's contents are in version 1 of the DTS
syntax, which has replaced a now-obsolete "version 0".

The tree has three *nodes*:

#. A root node: ``/``
#. A node named ``a-node``, which is a child of the root node
#. A node named ``a-sub-node``, which is a child of ``a-node``

.. _dt-node-labels:

Nodes can be given *labels*, which are unique shorthands that can be used to
refer to the labeled node elsewhere in the devicetree. Above, ``a-sub-node``
has label ``subnode_label``.

Devicetree nodes have *paths* identifying their locations in the tree. Like
Unix file system paths, devicetree paths are strings separated by slashes
(``/``), and the root node's path is a single slash: ``/``. Otherwise, each
node's path is formed by concatenating the node's ancestors' names with the
node's own name, separated by slashes. For example, the full path to
``a-sub-node`` is ``/a-node/a-sub-node``.

Devicetree nodes can also have *properties*. Properties are name/value
pairs. The values are simple byte arrays. Node ``a-sub-node`` has a property
named ``foo``, whose value is a 32-bit big-endian unsigned integer with value
3. The size and type of ``foo``\ 's value are implied by the enclosing angle
brackets (``<`` and ``>``) in the DTS. Refer to the Devicetree Specification
for a complete list of ways to write a property value in a DTS file.

In practice, devicetree nodes correspond to some hardware, and the node
hierarchy reflects the hardware's physical layout. For example, let's consider
a board with three I2C peripherals connected to an I2C bus controller on an SoC,
like this:

.. figure:: zephyr_dt_i2c_high_level.png
   :alt: representation of a board with three I2C peripherals
   :figclass: align-center

Nodes corresponding to the I2C bus controller and each I2C peripheral would be
present in this board's devicetree. Reflecting the hardware layout, the
devicetree's peripheral nodes would be children of the bus controller node.
Similar conventions exist for representing other types of hardware in
devicetree.

The DTS would look something like this:

.. code-block:: DTS

   /dts-v1/;

   / {
           soc {
                   i2c-bus-controller {
                           i2c-peripheral-1 {
                           };
                           i2c-peripheral-2 {
                           };
                           i2c-peripheral-3 {
                           };
                   };
           };
   };

Properties are used in practice to describe or configure the hardware the node
represents. For example, an I2C peripheral's node has a property whose value is
the peripheral's address on the bus.

Here's a tree representing the same example, but with real-world node
names and properties you might see when working with I2C devices.

.. figure:: zephyr_dt_i2c_example.png
   :figclass: align-center

   I2C devicetree example with real-world names and properties.
   Node names are at the top of each node with a gray background.
   Properties are shown as "name=value" lines.

This is the corresponding DTS:

.. code-block:: DTS

   /dts-v1/;

   / {
           soc {
                   i2c@40003000 {
                           compatible = "nordic,nrf-twim";
                           label = "I2C_0";
                           reg = <0x40003000 0x1000>;

                           apds9960@39 {
                                   compatible = "avago,apds9960";
                                   label = "APDS9960";
                                   reg = <0x39>;
                           };
                           ti_hdc@43 {
                                   compatible = "ti,hdc", "ti,hdc1010";
                                   label = "HDC1010";
                                   reg = <0x43>;
                           };
                           mma8652fc@1d {
                                   compatible = "nxp,fxos8700", "nxp,mma8652fc";
                                   label = "MMA8652FC";
                                   reg = <0x1d>;
                           };
                   };
           };
   };

.. _dt-unit-address:

In addition to showing more realistic names and properties, the above example
introduces a new devicetree concept: unit addresses. Unit addresses are the
parts of node names after an "at" sign (``@``), like ``40003000`` in
``i2c@40003000``, or ``39`` in ``apds9960@39``. Unit addresses are optional:
the ``soc`` node does not have one.

Some more details about unit addresses and important properties follow.

Unit address examples
*********************

In devicetree, unit addresses give a node's address in the
address space of its parent node. Here are some example unit addresses for
different types of hardware.

Memory-mapped peripherals
    The peripheral's register map base address.
    For example, the node named ``i2c@40003000`` represents an I2C controller
    whose register map base address is 0x40003000.

I2C peripherals
    The peripheral's address on the I2C bus.
    For example, the child node ``apds9960@39`` of the I2C controller
    in the previous section has I2C address 0x39.

SPI peripherals
    An index representing the peripheral's chip select line number.
    (If there is no chip select line, 0 is used.)

Memory
    The physical start address.
    For example, a node named ``memory@2000000`` represents RAM starting at
    physical address 0x2000000.

Memory-mapped flash
    Like RAM, the physical start address.
    For example, a node named ``flash@8000000`` represents a flash device
    whose physical start address is 0x8000000.

Flash partitions
    The start offset of the partition within its flash device.
    For example, take this flash device and its partitions:

    .. code-block:: DTS

        flash@8000000 {
            /* ... */
            partitions {
                    partition@0 { /* ... */ };
                    partition@20000 {  /* ... */ };
                    /* ... */
            };
        };

    The node named ``partition@0`` has offset 0 from the start of its flash
    device, so its base address is 0x8000000. Similarly, the base address of
    the node named ``partition@20000`` is 0x8020000.

.. _dt-important-props:

Important properties
********************

Some important properties are:

compatible
    Says what kind of device the node represents. The recommended format is
    ``"manufacturer,device"``, like ``"avago,apds9960"``, or a sequence of
    these, like ``"ti,hdc", "ti,hdc1010"``. The file
    :zephyr_file:`dts/bindings/vendor-prefixes.txt` contains a list of accepted
    ``manufacturer`` prefixes.

    It is also sometimes a value like ``gpio-keys``, ``mmio-sram``, or
    ``fixed-clock`` when the hardware's behavior is generic.

    The build system uses the compatible property to find the right
    :ref:`bindings <dt-bindings>` for the node.

label
    The device's name according to Zephyr's :ref:`device_model_api`. The value
    can be passed to :c:func:`device_get_binding()` to retrieve the
    corresponding driver-level :ref:`struct device* <device_struct>`. This
    pointer can then be passed to the correct driver API by application code to
    interact with the device. For example, calling
    ``device_get_binding("I2C_0")`` would return a pointer to a device
    structure which could be passed to :ref:`I2C API <i2c_api>` functions like
    :c:func:`i2c_transfer()`. The generated C header will also contain a macro
    which expands to this string.

reg
    Information used to address the device. This could be a memory-mapped I/O
    address range (as with ``i2c@40003000``\ 's reg property), an I2C bus
    address (as with ``apds9960@39`` and its devicetree siblings), a SPI chip
    select line, or some other value depending on the kind of device the node
    represents.

    Unlike a node's unit address, which is a simple number, the reg property is
    an array of 32-bit unsigned integers. This is often used to describe the
    size of a register map. In the case of the ``i2c@40003000`` node above,
    ``reg = <0x40003000 0x1000>;`` means the register map occupies 0x1000 bytes
    in the memory map.

.. _devicetree-in-out-files:

Input and output files
**********************

This section describes the input and output files shown in the figure at the
:ref:`top of this introduction <devicetree-intro>` in more detail.

.. figure:: zephyr_dt_inputs_outputs.svg
   :figclass: align-center

   Devicetree input (green) and output (yellow) files

There are four "types" of devicetree files:

- sources (``.dts``)
- includes (``.dtsi``)
- overlays (``.overlay``)
- bindings (``.yaml``)

The devicetree files inside the :file:`zephyr` directory look like this::

  boards/<ARCH>/<BOARD>/<BOARD>.dts
  dts/common/skeleton.dtsi
  dts/<ARCH>/.../<SOC>.dtsi
  dts/bindings/.../binding.yaml

Generally speaking, every supported board has a :file:`BOARD.dts` file
describing its hardware. For example, the ``reel_board`` has
:zephyr_file:`boards/arm/reel_board/reel_board.dts`.

:file:`BOARD.dts` includes one or more ``.dtsi`` files. These ``.dtsi`` files
describe the CPU or system-on-chip Zephyr runs on, perhaps by including other
``.dtsi`` files. They can also describe other common hardware features shared by
multiple boards. In addition to these includes, :file:`BOARD.dts` also describes
the board's specific hardware.

The :file:`dts/common` directory contains :file:`skeleton.dtsi`, a minimal
include file for defining a complete devicetree. Architecture-specific
subdirectories (:file:`dts/<ARCH>`) contain ``.dtsi`` files for CPUs or SoCs
which extend :file:`skeleton.dtsi`.

The C preprocessor is run on all devicetree files to expand macro references,
and includes are generally done with ``#include <filename>`` directives, even
though DTS has a ``/include/ "<filename>"`` syntax.

:file:`BOARD.dts` can be extended or modified using *overlays*. Overlays are
also DTS files; the :file:`.overlay` extension is just a convention which makes
their purpose clear. Overlays adapt the base devicetree for different purposes:

- Zephyr applications can use overlays to enable a peripheral that is disabled
  by default, select a sensor on the board for an application specific purpose,
  etc. Along with :ref:`kconfig`, this makes it possible to reconfigure the
  kernel and device drivers without modifying source code.

- Overlays are also used when defining :ref:`shields`.

The build system automatically picks up :file:`.overlay` files stored in
certain locations. It is also possible to explicitly list the overlays to
include, via the :makevar:`DTC_OVERLAY_FILE` CMake variable. See
:ref:`application_dt` and :ref:`important-build-vars` for details.

The build system combines :file:`BOARD.dts` and any :file:`.overlay` files by
concatenating them, with the overlays put last. This relies on DTS syntax which
allows merging overlapping definitions of nodes in the devicetree. See
:ref:`dt_k6x_example` for an example of how this works (in the context of
``.dtsi`` files, but the principle is the same for overlays). Putting the
contents of the :file:`.overlay` files last allows them to override
:file:`BOARD.dts`.

:ref:`dt-bindings` (which are YAML files) are essentially glue. They describe
the contents of devicetree sources, includes, and overlays in a way that allows
the build system to generate C macros usable by device drivers and
applications. The :file:`dts/bindings` directory contains bindings.

These files in the build directory can be useful as a debugging aid when
working with devicetree:

build/zephyr/<BOARD>.dts.pre.tmp
   The preprocessed and concatenated DTS sources

build/zephyr/zephyr.dts
   The final merged devicetree. This file is specifically output as a
   debugging aid, and is unused otherwise.

.. _dt-scripts:

The following libraries and scripts, located in :zephyr_file:`scripts/dts/`,
are used to generate C headers from the devicetree and its bindings. Note that
the source code has extensive comments and documentation.

:zephyr_file:`dtlib.py <scripts/dts/dtlib.py>`
    A low-level DTS parsing library.

:zephyr_file:`edtlib.py <scripts/dts/edtlib.py>`
    A library layered on top of dtlib that uses bindings to interpret
    properties and give a higher-level view of the devicetree. Uses dtlib to do
    the DTS parsing.

:zephyr_file:`gen_defines.py <scripts/dts/gen_defines.py>`
    A script that uses edtlib to generate C preprocessor macros from the
    devicetree and bindings.

The output from :file:`gen_defines.py` is stored in the build directory as
:file:`build/zephyr/include/generated/devicetree_unfixed.h`.

In addition to the Python code above, the standard ``dtc`` (devicetree
compiler) tool is also run on the final devicetree if it is installed on your
system. This is just to catch any errors or warnings it generates. The output
is unused. Boards may need to pass ``dtc`` additional flags, e.g. for warning
suppression. Board directories can contain a file named
:file:`pre_dt_board.cmake` which configures these extra flags, like this:

.. code-block:: cmake

   list(APPEND EXTRA_DTC_FLAGS "-Wno-simple_bus_reg")

Zephyr currently uses :file:`dts_fixup.h` files to rename macros in
:file:`devicetree_unfixed.h` to names that are currently in use by C code. The
build system looks for fixup files in the :file:`zephyr/boards/` and
:file:`zephyr/soc/` directories by default. Any :file:`dts_fixup.h` files are
concatenated and stored in the build directory as
:file:`build/zephyr/include/generated/devicetree_fixups.h`.

Fixup files exist for historical reasons. New code should generally avoid them.

To reference macros generated by :file:`gen_defines.py` from C, include
:file:`devicetree.h`. This file is :zephyr_file:`include/devicetree.h` in the
zephyr repository; it is not a generated file. It includes the generated
:file:`include/devicetree_unfixed.h` and :file:`include/devicetree_fixups.h`
files.

.. warning::

   Do not include the generated C headers from the build directory directly.
   Use :file:`devicetree.h` instead.
