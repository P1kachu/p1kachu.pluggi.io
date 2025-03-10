## Module excavation ?

Use concolic analysis to explore kernel modules and get informations about
their IOCTLs
<!-- .element: class="fragment"-->


---


## What is an IOCTL ?

```c
long random_ioctl(int fd, unsigned int cmd, unsigned long arg);
```

* A syscall to get custom operations on a resource<!-- .element: class="fragment"-->
* Device specific commands, code or specs needed<!-- .element: class="fragment"-->
* Unavailable for private drivers<!-- .element: class="fragment"-->


---


## But, why ?


---


#### Check if headers and IOCTLs match

```c
// linux/include/uapi/linux/firewire-cdev.h

#define FW_CDEV_IOC_GET_INFO           _IOWR('#', 0x00, struct fw_cdev_get_info)
#define FW_CDEV_IOC_SEND_REQUEST        _IOW('#', 0x01, struct fw_cdev_send_request)
#define FW_CDEV_IOC_ALLOCATE           _IOWR('#', 0x02, struct fw_cdev_allocate)
#define FW_CDEV_IOC_DEALLOCATE          _IOW('#', 0x03, struct fw_cdev_deallocate)
#define FW_CDEV_IOC_SEND_RESPONSE       _IOW('#', 0x04, struct fw_cdev_send_response)
#define FW_CDEV_IOC_INITIATE_BUS_RESET  _IOW('#', 0x05, struct fw_cdev_initiate_bus_reset)
#define FW_CDEV_IOC_ADD_DESCRIPTOR     _IOWR('#', 0x06, struct fw_cdev_add_descriptor)
#define FW_CDEV_IOC_REMOVE_DESCRIPTOR   _IOW('#', 0x07, struct fw_cdev_remove_descriptor)
#define FW_CDEV_IOC_CREATE_ISO_CONTEXT _IOWR('#', 0x08, struct fw_cdev_create_iso_context)
#define FW_CDEV_IOC_QUEUE_ISO          _IOWR('#', 0x09, struct fw_cdev_queue_iso)
#define FW_CDEV_IOC_START_ISO           _IOW('#', 0x0a, struct fw_cdev_start_iso)
#define FW_CDEV_IOC_STOP_ISO            _IOW('#', 0x0b, struct fw_cdev_stop_iso)
```
<!-- .element: style="font-size: 0.40em;" -->


---

### IOCTL commands contain data

```c
// linux/include/uapi/linux/firewire-cdev.h
#define FW_CDEV_IOC_GET_INFO           _IOWR('#', 0x00, struct fw_cdev_get_info)

```
<!-- .element: style="font-size: 0.40em;" -->


```c
// linux/include/uapi/asm-generic/ioctl.h
#define _IOC(dir,type,nr,size) \
        (((dir)  << _IOC_DIRSHIFT) | \
         ((type) << _IOC_TYPESHIFT) | \
         ((nr)   << _IOC_NRSHIFT) | \
         ((size) << _IOC_SIZESHIFT))

#ifndef __KERNEL__
#define _IOC_TYPECHECK(t) (sizeof(t))
#endif

/* used to create numbers */
#define _IO(type,nr)            _IOC(_IOC_NONE,(type),(nr),0)
#define _IOR(type,nr,size)      _IOC(_IOC_READ,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOW(type,nr,size)      _IOC(_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
#define _IOWR(type,nr,size)     _IOC(_IOC_READ|_IOC_WRITE,(type),(nr),(_IOC_TYPECHECK(size)))
```
<!-- .element: style="font-size: 0.40em;" -->


---


#### Still doesn't tell us why...


* To find bugs <!-- .element: class="fragment"-->
* To find vulnerabilities (Yay)<!-- .element: class="fragment"-->
* To discover IOCTLs from private drivers<!-- .element: class="fragment"-->


---

#### And, as a bonus

Experience and challenge this kind of analysis in a new context

A.K.A not in a userland CTF exercise


---

## The peeler: steps

* Find the functions accurately<!-- .element: class="fragment"-->
* Find which commands are valid<!-- .element: class="fragment"-->
* Find a way to determine the type of 'arg' <!-- .element: class="fragment"-->

---


## angr
![Image](https://avatars1.githubusercontent.com/u/12520807?v=3&s=200)
<!-- .element: style="max-height: 200px;"-->
![Image](https://ctftime.org/media/team/shellphish_logo_400px.png)
<!-- .element: style="max-height: 200px;"-->

Framework developped by the UC Santa Barbara's Computer Security Lab, and their
associated CTF team, Shellphish.


---


#### What is it ?

> angr is a framework for analyzing binaries. It focuses on both
> static and dynamic symbolic ("concolic") analysis, making it
> applicable to a variety of tasks.
<!-- .element: style="font-size: 0.60em;" -->

Participated in the DARPA CGC (Autonomous Hacking) - One of the 7 team
qualified for the finals

Submodules: CLE, claripy, simuvex...


---


### Concolic ?

**Conc**rete execution + Symb**olic** execution
* Concrete execution: Program being executed <!-- .element: class="fragment" -->
* Symbolic execution allows at a time T to determine for a branch all conditions
necessary to take the branch or not
<!-- .element: class="fragment" -->


---

## Example

```c
int example(int x, int y)
{
    int x = i1;
    int y = i2;

    if (x > 80) {
        if (x == 256)
            return True;
    } else {
        x = 0;
        y = 0;
    }
    return False;
}
```


---


## Gives us
![Image](rsc/concolic.svg)
<!-- .element: style="max-height: 500px;"-->


---

## Practical example

Defcon Quals 2016 - babyre

![Image](http://p1kachu.pluggi.fr/assets/content/defcon2016_graph.png)
<!-- .element: style="max-height: 400px;"-->


---

Solved in 5 minutes with angr:

```python
main           = 0x4025e7

p = angr.Project('baby-re')
init = p.factory.blank_state(addr=main)
```
<!-- .element: class="fragment" -->
<!-- .element: style="font-size: 0.40em;" -->

```python
# Taken from IDA's xrefs
scanf_off = [0x4d, 0x85, 0xbd, 0xf5, 0x12d, 0x165, 0x19d, 0x1d5,
             0x20d, 0x245, 0x27d, 0x2b5, 0x2ed]

def scanf(state):
    state.mem[state.regs.rsi:] = state.se.BVS('c', 8)

for o in scanf_off:
        p.hook(main + o, func=scanf, length=5)
```
<!-- .element: class="fragment" -->
<!-- .element: style="font-size: 0.40em;" -->

```python
pgp = p.factory.path_group(init, threads=8)

win            = 0x4028e9
fail           = 0x402941
ex = pgp.explore(find=(win), avoid=(fail))
```
<!-- .element: class="fragment" -->
<!-- .element: style="font-size: 0.40em;" -->

```python
s = ex.found[0].state

flag_addr      = 0x7fffffffffeff98 # First rsi from scanf
flag = s.se.any_str(s.memory.load(flag_addr, 50))
print("The flag is '{0}'".format(flag))
```
<!-- .element: class="fragment" -->
<!-- .element: style="font-size: 0.40em;" -->


---

### So ?

* It seems to do everything we ask for
* Good results in CTF
* Most of the work has been put in the ELF handling


---



# BUT

### (Yes, there is a but...)
<!-- .element: class="fragment" -->

##### (again...)
<!-- .element: class="fragment" -->


---

> Apparently it doesn't like kernel modules,
> you need to write a custom loader
>
> -- Gaby

![Image](http://i.giphy.com/gKjmphDeV05k4.gif)
<!-- .element: class="fragment" -->
<!-- .element: style="max-height: 280px;" -->


---


### Problems

* Object files (modules) are different from executables
* Relocations had to be done


---


### Relocations

* References to symbols in other sections
* Need to be resolved at link time


---

#### Example

```ruby
;; x.o

.text:
    f:
        call external_func      ;; Relocation to external func
        lea eax, inter_section, ;; Inter section relocation
        ret

.data:
    inter_section:
        .long 12
```

```ruby
;; y.o

.text:
    main:
        call f                   ;; Inter object relocation
```

---


# Let's explore


---


Peeler behavior overview:

```C
ioctls = find_ioctls("peel_me_sensually.bin");
for (ioctl in ioctls)
{
    endpoints = find_endpoints(ioctl);
    ex = explorer();
    for (endpoint in endpoints)
    {
        paths = get_paths(ex, ioctl.entry, endpoint);
        for (path in paths)
        {
            if (get_ret_val(path) > -1)
                do_stuff(path);
        }
    }
}
```

---


```C
int my_false_ioctl(int fd, unsigned long cmd, void* arg) {

    int ret = -1;

    switch (cmd) {
    	case 0xcafe:
	        ret = 1 * 2 + 98 - 3000;

	        if (ret + fd - 23 + cmd == 0xa110c)
	    	    ret = 1;
    }
    return ret;
}
```
gives us

```C
Path from 0x4005dc to 4006a7
  Required conditions (constraints):
    <Bool reg_40_5_64 == 0xcafe>
    <Bool (reg_40_5_64 + SignExt(32, ((reg_48_4_64 + 0xfffff4ac) - 0x17))) == 0xa110c>
  Simplified: <Bool (reg_40_5_64 == 0xcafe)  \
  		       && (reg_48_4_64 == 0x95179) \
		         && ((0xfffff495 + reg_48_4_64[31:0])[31:31] == 0))>
  return value: 0x1
```
<!-- .element: style="font-size: 0.40em;" -->


---

### Minor fixes

The intra-block address patch

![Image](https://i.imgur.com/VXhHWFu.png)
<!-- .element: style="max-height:650px;" -->


---


### Hey, it wor-- ... Wait a minute.
![Image](https://i.imgur.com/UsGvdsb.png)
<!-- .element: style="max-height:500px;" -->
<!-- .element: class="fragment" -->
![Image](https://i.imgur.com/tKvcIrF.png)
<!-- .element: style="max-height:500px;" -->
<!-- .element: class="fragment" -->

(drm.ko)
<!-- .element: class="fragment" -->
<!-- .element: style="font-size: 0.40em;" -->


---

#### drm_mode_atomic_ioctl

```console
p1kachu@GreenLabOfGazon:src$ ./pyfinder.py drm.ko -f drm_mode_atomic_ioctl -q
[ ]   INFOS   Peeling drm's ioctls

[ ]   INFOS   Analyzing function drm_mode_atomic_ioctl at 0x421b30
[ ]   INFOS   Launching path_group explorer
[ ]   INFOS   Explorer: <PathGroup with 1 deadended, 1 found>

[ ]   INFOS   Analyzing 1 found paths
[ ]   INFOS   Path from 0x421b30 to 0x421f12L (1/1)
[ ]   INFOS   Return value would be 0xffffffeaL - Skipping

[ ]   INFOS   Analyzing 1 deadended paths
[ ]   INFOS   Path from 0x421b30 to 0x421ba7L (1/1)
[-]    FAIL   Something went wrong in se.min/max: Unsat Error
[ ]   INFOS   End of analysis
```
<!-- .element: style="font-size: 0.40em;" -->


---

#### drm_compat_ioctl

```console
p1kachu@GreenLabOfGazon:src$ ./pyfinder.py drm.ko -f drm_compat_ioctl -q
[ ]   INFOS   Peeling drm's ioctls

[ ]   INFOS   Analyzing function drm_compat_ioctl at 0x422490
[ ]   INFOS   Launching path_group explorer
[ ]   INFOS   Explorer: <PathGroup with 5 deadended, 2 active, 1 found>

[ ]   INFOS   Analyzing 1 found paths
[ ]   INFOS   Path from 0x422490 to 0x4224bdL (1/1)
[ ]   INFOS   Return value would be 0xffffffffffffffedL - Skipping

[ ]   INFOS   Analyzing 5 deadended paths
[ ]   INFOS   Path from 0x422490 to 0x405b44L (1/5)
[-]    FAIL   Something went wrong in se.min/max: Unsat Error
[ ]   INFOS   Path from 0x422490 to 0x4059f5L (2/5)
[-]    FAIL   Something went wrong in se.min/max: Unsat Error
[ ]   INFOS   Path from 0x422490 to 0x423e4eL (3/5)
[ ]   INFOS   Return value would be 0xfffffff2L - Skipping
[ ]   INFOS   Path from 0x422490 to 0x405b44L (4/5)
[-]    FAIL   Something went wrong in se.min/max: Unsat Error
[ ]   INFOS   Path from 0x422490 to 0x4059f5L (5/5)
[-]    FAIL   Something went wrong in se.min/max: Unsat Error
[ ]   INFOS   Explorer: <PathGroup with 2 deadended>
```
<!-- .element: style="font-size: 0.36em;" -->

---


#### __kstrtab_drm_ioctl_permit

```console
[ ]   INFOS   Analyzing function __kstrtab_drm_ioctl_permit at 0x4399ce
Traceback (most recent call last):
  File "./pyfinder.py", line 201, in <module>
      recover_function(f, cfg, addr)
  File "/home/p1kachu/peeling-ioctls/src/excavator.py", line 195, in recover_function
      ins = blk.capstone.insns[last_ins]
IndexError: list index out of range
```
<!-- .element: style="font-size: 0.40em;" -->


---


### What now ?

* We need to enhance and strengthen the peeler
* angr cannot work without some human pre-work
* How to save resources (time and memory) ?
  * Automate verifications
  * Discard useless stuff
  * Analyze interesting functions only


---


## Find IOCTLs smartly

#### How do we do that ?


---

### IOCTL registration processus

* Create a struct file_operations
  * Multiple function pointers
  * Used to register operations on the device
* Load the structure in memory (using a register function)
* Classic operations will now be handled by these functions


```c
static const struct file_operations i8k_fops = {
        .owner          = THIS_MODULE,
        .open           = i8k_open_fs,
        .read           = seq_read,
        .llseek         = seq_lseek,
        .release        = single_release,
        .unlocked_ioctl = i8k_ioctl,
};
```

---


#### Legend

```console
For the next slides, please refer to this legend:
- fops           : file_operations struct containing our ioctl
- register_ioctl : register function that will load fops
- call_me_addr   : address of 'call register_ioctl'
- Caller         : function containing call_me_addr
```


---


### Goal: Find fops

```c
static struct file_operations fops = {
    .owner = THIS_MODULE,
    .unlocked_ioctl = (void*)my_ioctl,
    .compat_ioctl = (void*)my_ioctl
};
```
Data of interest: Its address in memory.
<!-- .element: class="fragment" -->

---


### Look for register_ioctl

Iterate over imported symbols to look for one of these:

```c
register_chrdev(DRM_MAJOR, "drm", &drm_stub_fops);
```

```python
register_functions = [
                      '__register_chrdev',
                      'misc_register',
                      'cdev_init'
]

```

Data of interest: Exact address of 'call register_ioctl' (call_me_addr).<!-- .element: class="fragment" -->


---


### Find which function calls register_ioctl (Caller)

```python
"""
- The 'call register_ioctl' will always be in the init function.
- The init function will always be called init_module.
"""
```

```python
"""
- The 'call register_ioctl' will always be in the init function.
- The init function will always be called init_module.

EDIT : No, and no.
"""

for _, sym in elf.symbols_by_addr.iteritems():
    bottom = sym.rebased_addr
    top    = sym.rebased_addr + sym.size
    if register_ioctl > bottom and register_ioctl <= top:
        Caller = sym.name
```
<!-- .element: class="fragment" -->
Data of interest: Entry point of Caller.<!-- .element: class="fragment" -->


---


### And now ?

```markdown
We:
* have Caller's entry point
* have register_ioctl's call address (call_me_addr, in Caller)
* know that when register_ioctl is called, fops is in a register
```

```markdown
So we:
* Launch a path explorer from Caller's entry point to call_me_addr
* Break just before the call
* Analyze the passed arguments to get fops address
```
<!-- .element: class="fragment" -->


---


### Problems

* Very long Callers
* Lots of unresolved functions
* Memory and time consuming
* Calling 'conventions'

![Image](http://i.giphy.com/lngw0pFJezv9vWdDq.gif)
<!-- .element: class="fragment" -->
<!-- .element: style="max-height:200px;" -->

---


### Let's try to be clever

Assignations and call usually are in the same block

![Image](http://i.imgur.com/aoL9WaT.png)
<!-- .element: style="max-height:200px;" --><!-- .element: class="fragment" -->
![Image](http://i.imgur.com/lt5PZAR.png)
<!-- .element: style="max-height:200px;" --><!-- .element: class="fragment" -->
![Image](http://i.imgur.com/0Uexxja.png)
<!-- .element: style="max-height:200px;" --><!-- .element: class="fragment" -->


---



* Create a CFG of Caller
* Find the basic block containing call_me_addr<!-- .element: class="fragment" -->
* Path explorer from the beginning of the block only<!-- .element: class="fragment" -->

![Image](http://i.imgur.com/0Uexxja.png)
<!-- .element: style="max-height:200px;" --><!-- .element: class="fragment" -->


---


Right.

It doesn't work. <!-- .element: class="fragment" -->


---

#### Troubles with incomplete CFG


![Image](https://i.imgur.com/NBNDACV.png)



---


![Image](https://i.imgur.com/7vmjfFJ.png)


---


Problems with relative calls, unresolved symbols, symbolic memory...

![Image](http://i.giphy.com/Lvand4cUuA6xG.gif)<!-- .element: class="fragment" -->


---

## Another way: DWARF debug infos

If present, DWARF infos *can not* fail

Iterate over them, find fops in memory, and get the IOCTLs


---



## Best effort strategy

```markdown
1. look in debug infos (DWARF)
    * Accurate and fast
    * Need to have access to the source code

2. Fallback on the file_operations structure
    * Slow
    * Requires an angr explorer and CFG for itself
    * Often fails at some point

3. Fallback on symbols names
    * Some IOCTLs aren't named like that
    * More functions to analyze
```
<!-- .element: class="fragment" -->

---


## What's left to do ?

![Image](http://i.giphy.com/OCdsZb1Er6aeQ.gif)
<!-- .element: class="fragment" -->

```markdown
* Get infos about the *arg* parameter
    * What's its type ?
    * Which operations are applied on it ?
```
<!-- .element: class="fragment" -->

---


## Possible improvement

* Efficiency boost (merge paths, discard others, ...)
* Allow user input for testing
* Sanity checking by parsing headers


---

## Sum up

* Not usable for big/complicated modules
* Would need more layers of fallbacks
* Everything is too unstable to be used at once

However, still interesting to see angr on real life problems

---


* More details:
    * [Linux Device Drivers - Chapter 6](https://lwn.net/Kernel/LDD3/)
	* http://github.com/angr
    * [SoK: (State of) The Art of War: Offensive Techniques in Binary Analysis](https://www.cs.ucsb.edu/~vigna/publications/2016_SP_angrSoK.pdf) (UCSB)
	* [Binary analysis - Concolic Execution](http://shell-storm.org/blog/Binary-analysis-Concolic-execution-with-Pin-and-z3/) (Jonathan Salwan)


---


Thank you

![Image](http://i.giphy.com/4ECepQbplGd0I.gif)

[p1kachu@lse.epita.fr](mailto:p1kachu@lse.epita.fr) - [@0xP1kachu](http://twitter.com/0xP1kachu)
