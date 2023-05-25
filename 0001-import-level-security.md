# Security in MicroPython

**TL;DR: See practical example at the end.**

MicroPython allows applications to be written fast in a bare-metal environment
where no kernel security mechanism is present.

Existing operating system do not provide security inside of an application
itself, where an exposed interface (i.e. web server) is very close to a
defended interface (i.e. motor control, sensor data, private storage...).

The proposal is to bring a function that remove the ability to the current
module to import anything more.

Challenge
--------------------------------------------------------------------------------
Its support got extended from the PyBoard to an many other boards, some of
which support TCP/IP stack, and getting exposed to WAN or a LAN directly. This
opens new challeng regarding security of the devices, as a new bug in
MicroPython can turn into a CVE.

Multiple significant projects are in their way to use MicroPython as a
foundation and bringing security to MicroPython is an opportunity to bring more
security to the surrounding systems as well.

Increasingly large projects start using MicroPython, which includes Quectel with
[QuecPython](https://python.quectel.com/), Adafruit with Circuit Python,
and numerous projects with a MicroPython firmware.

Discussion: avoid an illusion of security
--------------------------------------------------------------------------------
Operating Systems offer strong protection across multiple processes and defend
the kernel like a dungeon. This allows unrelated programs to run, crash, and be
exploited while other can safely continue their execution.

For embedded systems, though, it is likely that a single process will provide
the main function of the device: memory isolation does not help if there is
only a single big memory pool: if all the precious data is contained in the
same memory pool as the part exposed to Internet.

Web applications are be frequently compromised, even though the rest of the
system remains safe. The operating system security mechanism does not benefit
the application.

Discussion: memory safety as single point of security
--------------------------------------------------------------------------------
Memory safety is a good thing. It relies on a high-quality source code to
protect everything else.

If a CVE involving an exploit breaking MicroPython memory safety may happens,
it is preferable to not make this mechanism the only one providing security.

It also does not protect against several kind of attacks such as writing
python to storage and executing it, etc.

Proposal: bring tools to implement security at application level
--------------------------------------------------------------------------------
What is implied is that no single mechanism is a silver bullet, in particular
when the dangerous wolf and fragile lamb are in the same pot.

In order to make efficient use of hardware security mechanisms when they are
available would be to wrap them up, and provide an excessively convenient
interface to the user who will be able to split its "dangerous zone" exposed
to the network, and the "safe area" with the raw hardware access.

Ultimately, the desired mechanism is to allow to drop privileges.

Python implementation: pledge(2)
--------------------------------------------------------------------------------
Modeled after OpenBSD's [pledge(2)](https://man.openbsd.org/pledge), it would be
possible to have some builtin function to disallow importing anything more on
the current module. This would already prevent the access to i.e. SPI, network,
filesystem, etc.

With the synta `from x import y`, it would further be possible to very
fine-tune the access to specific functions only.

This is also likely straightforward to implement: a single flag placed at the
right location in the source allows to prevent any further import.

Python implementation: unveil(2)
--------------------------------------------------------------------------------
Because some python code might need to open files with dynamic names, it can be
abused to store code onto the filesystem, which can then eventually be executed 

To allow such code to be written in a secure way, it is possible to implement a
similar approach to [unveil(2)](https://man.openbsd.org/unveil): restrict the
paths that can be opened in each different mode (read/write/browse/create/...).

This could allow the builtin functions such as `open()` which are always
available to be further restricted.

Disabling some of these builtin functions could also be necessary.

C source implementation: isolating modules to isolate drivers
--------------------------------------------------------------------------------
This security mechanism allows to restrict the access to several *drivers*,
such as an SPI driver, an I2C driver... even though they are modules in
disguise.

The MicroPython C source has a good *driver isolation* that Linux lacks to some
extent [1]:

MicroPython modules do not need to have any header to register themself to the
rest of the system, as the Makefile toolchain greps for module definition
everywhere, which scans the system for configured modules and there is hence
no need for any header.

This lack of header encourages to have all communication with modules happen
not a C source level, but rather within the memory-safer python language:
C modules do not call other C module's functions except through the python
interface.

Thanks to this, there is a good amount of isolation between micropython
"drivers" (modules), and so for all ports thanks to MicroPython C programming
practices.

This further strenghten the discussed isolation mechanism (a way to block
"import") as a strong level of restriction.

> [1]: With performance no longer being the main roadblock, the complexity of
> isolating device drivers has become the main challenge. Device drivers and
> kernel extensions are developed in a shared memory environment in which the
> state shared between the kernel and the driver is mixed in a complex
> hierarchy of data structures, making it difficult for programmers to ensure
> that the shared state is synchronized correctly. In this paper, we present
> KSplit, a new framework for isolating [...]
> <https://www.usenix.org/conference/osdi22/presentation/huang-yongzhe>

How to strengthen things at hardware level?
--------------------------------------------------------------------------------
Adding a software-level barrier helps, but could be bypassed in the event of a
bug or if something was not planned.

How could hardware level mechanisms help further more?
Would it be possible to enforce this barrier from the hardware itself?

C modules would have their own address range, this would allow MPU to take
effect based on the current list of imported module?

Would it be possible to have access to specific hardware registers only made
available to one or multiple C modules and prevent any access from any other
memory region (such as python or direct memory read/write)?

This might go beyond the scope of this single import barrier proposed, although
could come as a complement.

Example: restrict I2C addresses and values that can be accessed
--------------------------------------------------------------------------------
A practical example would be

In `i2c_interface.py`

```python
import i2c_module_from_sdk_implemented_in_c as i2c

MY_DEVICE_ADDR = 0x12
MY_REGISTER_ADDR = 0xF7
MY_SAFE_VALUES = range(100)

def write(device_addr, register_addr, value_to_write):
    if device_addr != MY_DEVICE_ADDR:
        raise ValueError("invalid device address")
    if register_addr != MY_REGISTER_ADDR:
        raise ValueError("invalid register address")
    if value_to_write not in MY_SAFE_VALUES:
        raise ValueError("invalid value")
    i2c.write(device_addr, bytearray(register_addr, value_to_write))
```

In `dangerously_exposed_applications.py`

```python
import i2c_interface
import network_server
disable_imports()

while True:
    # blocking
    request = network_server.handle_request():
    device_addr = 0
    register_addr = 0
    value_to_write = 0

    # a lot of error-prone network request parsing and parameter validation

    i2c_interface.write(device_addr, register_addr, value_to_write)
```
