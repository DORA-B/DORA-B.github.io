---
title: Modify python pwnlib to enable atampt assemble
date: 2024-01-28
categories: [Assembly]
tags: [assembly,python]     # TAG names should always be lowercase
---
Problem statement: Cannot assemble AT&T assembly code because of pwn's default is Intel assemble
================================================================================================

# Errors encountered

```bash
[ERROR] There was an error running ['/usr/bin/x86_64-linux-gnu-as', '-32', '-o', '/tmp/pwn-asm-dg13b0er/step2', '/tmp/pwn-asm-dg13b0er/step1']:
    It had the exitcode 1.
    It had this on stdout:
    /tmp/pwn-asm-dg13b0er/step1: Assembler messages:
    /tmp/pwn-asm-dg13b0er/step1:8: Error: no such instruction: `movl $0x1,%eax'
    /tmp/pwn-asm-dg13b0er/step1:9: Error: no such instruction: `movl $0x0,%ebx'
    /tmp/pwn-asm-dg13b0er/step1:10: Error: operand size mismatch for `int'
  
[ERROR] An error occurred while assembling:
       1: .section .shellcode,"awx"
       2: .global _start
       3: .global __start
       4: _start:
       5: __start:
       6: .intel_syntax noprefix
       7: .p2align 0
       8:     movl $0x1, %eax
       9:     movl $0x0, %ebx
      10:     int $0x80
    Traceback (most recent call last):
      File "/home/qingchen/PyEnv/ml-isa-env/lib/python3.8/site-packages/pwnlib/asm.py", line 713, in asm
        _run(assembler + ['-o', step2, step1])
      File "/home/qingchen/PyEnv/ml-isa-env/lib/python3.8/site-packages/pwnlib/asm.py", line 431, in _run
        log.error(msg, *args)
      File "/home/qingchen/PyEnv/ml-isa-env/lib/python3.8/site-packages/pwnlib/log.py", line 439, in error
        raise PwnlibException(message % args)
    pwnlib.exception.PwnlibException: There was an error running ['/usr/bin/x86_64-linux-gnu-as', '-32', '-o', '/tmp/pwn-asm-dg13b0er/step2', '/tmp/pwn-asm-dg13b0er/step1']:
    It had the exitcode 1.
    It had this on stdout:
    /tmp/pwn-asm-dg13b0er/step1: Assembler messages:
    /tmp/pwn-asm-dg13b0er/step1:8: Error: no such instruction: `movl $0x1,%eax'
    /tmp/pwn-asm-dg13b0er/step1:9: Error: no such instruction: `movl $0x0,%ebx'
    /tmp/pwn-asm-dg13b0er/step1:10: Error: operand size mismatch for `int'
```

> And where I call for this code is here:

```python
def test_pwn_asm(inst: str) -> bytes:
    from pwn import asm,context
    # context.update(arch='x86', os='linux', endian='little', bits=32)
    context.arch = 'i386'  # Example: set architecture to 32-bit x86
    context.os = 'linux'
    context.endian = 'little'
    context.bits = 32
    # Explicitly set the syntax to AT&T
    # context.syntax = 'att'
    # Assemble the instruction using AT&T syntax
    print(asm(inst))
def main():
    att_assembly_code = """
    movl $0x1, %eax
    movl $0x0, %ebx
    int $0x80
    """
    test_pwn_asm(att_assembly_code)
```

# Why and how to address this problem

IN pwnlib's asm.py, define asm function

```python
@LocalContext
def asm(shellcode, vma = 0, extract = True, shared = False):
    r"""asm(code, vma = 0, extract = True, shared = False, ...) -> str

    Runs :func:`cpp` over a given shellcode and then assembles it into bytes.

    To see which architectures or operating systems are supported,
    look in :mod:`pwnlib.context`.

    Assembling shellcode requires that the GNU assembler is installed
    for the target architecture.
    See :doc:`Installing Binutils </install/binutils>` for more information.

    Arguments:
        shellcode(str): Assembler code to assemble.
        vma(int):       Virtual memory address of the beginning of assembly
        extract(bool):  Extract the raw assembly bytes from the assembled
                        file.  If :const:`False`, returns the path to an ELF file
                        with the assembly embedded.
        shared(bool):   Create a shared object.
        kwargs(dict):   Any attributes on :data:`.context` can be set, e.g.set
                        ``arch='arm'``.

    Examples:

        >>> asm("mov eax, SYS_select", arch = 'i386', os = 'freebsd')
        b'\xb8]\x00\x00\x00'
        >>> asm("mov eax, SYS_select", arch = 'amd64', os = 'linux')
        b'\xb8\x17\x00\x00\x00'
        >>> asm("mov rax, SYS_select", arch = 'amd64', os = 'linux')
        b'H\xc7\xc0\x17\x00\x00\x00'
        >>> asm("mov r0, #SYS_select", arch = 'arm', os = 'linux', bits=32)
        b'R\x00\xa0\xe3'
        >>> asm("mov #42, r0", arch = 'msp430')
        b'0@*\x00'
        >>> asm("la %r0, 42", arch = 's390', bits=64)
        b'A\x00\x00*'
    """
    result = ''

    assembler = _assembler()
    linker    = _linker()
    objcopy   = _objcopy() + ['-j', '.shellcode', '-Obinary']
    code      = ''
    code      += _arch_header()
    code      += cpp(shellcode)

    log.debug('Assembling\n%s' % code)

    tmpdir    = tempfile.mkdtemp(prefix = 'pwn-asm-')
    step1     = path.join(tmpdir, 'step1')
    step2     = path.join(tmpdir, 'step2')
    step3     = path.join(tmpdir, 'step3')
    step4     = path.join(tmpdir, 'step4')

    try:
        with open(step1, 'w') as fd:
            fd.write(code)

        _run(assembler + ['-o', step2, step1])

        if not vma:
            shutil.copy(step2, step3)

        if vma or not extract:
            ldflags = _execstack(linker) + ['-o', step3, step2]
            if vma:
                ldflags += ['--section-start=.shellcode=%#x' % vma,
                            '--entry=%#x' % vma]
            elif shared:
                ldflags += ['-shared', '-init=_start']

            # In order to ensure that we generate ELF files with 4k pages,
            # and not e.g. 65KB pages (AArch64), force the page size.
            # This is a result of GNU Gold being silly.
            #
            # Introduced in commit dd58f409 without supporting evidence,
            # this shouldn't do anything except keep consistent page granularity
            # across architectures.
            ldflags += ['-z', 'max-page-size=4096',
                        '-z', 'common-page-size=4096']

            _run(linker + ldflags)

        elif open(step2,'rb').read(4) == b'\x7fELF':
            # Sanity check for seeing if the output has relocations
            relocs = subprocess.check_output(
                [which_binutils('readelf'), '-r', step2],
                universal_newlines = True
            ).strip()
            if extract and len(relocs.split('\n')) > 1:
                log.error('Shellcode contains relocations:\n%s' % relocs)
        else:
            shutil.copy(step2, step3)

        if not extract:
            return step3

        _run(objcopy + [step3, step4])

        with open(step4, 'rb') as fd:
            result = fd.read()

    except Exception:
        lines = '\n'.join('%4i: %s' % (i+1,line) for (i,line) in enumerate(code.splitlines()))
        log.exception("An error occurred while assembling:\n%s" % lines)
    else:
        atexit.register(lambda: shutil.rmtree(tmpdir))

    return result

```

And the most significant code in here is:

```python
assembler = _assembler() # find the assemle which form binutils in the system --> /usr/bin/$ARCH-*-as
linker    = _linker()
objcopy   = _objcopy() + ['-j', '.shellcode', '-Obinary']
code      = ''
code      += _arch_header() # Attention Here 
code      += cpp(shellcode)
```

in the function `_arch_header()`

```python
def _arch_header():
    prefix  = ['.section .shellcode,"awx"',
                '.global _start',
                '.global __start',
                '_start:',
                '__start:']
    headers = {
        'i386'  :  ['.intel_syntax noprefix', '.p2align 0'],
        'amd64' :  ['.intel_syntax noprefix', '.p2align 0'],
        'arm'   : ['.syntax unified',
                   '.arch armv7-a',
                   '.arm',
                   '.p2align 2'],
        'thumb' : ['.syntax unified',
                   '.arch armv7-a',
                   '.thumb',
                   '.p2align 2'
                   ],
        'mips'  : ['.set mips2',
                   '.set noreorder',
                   '.p2align 2'
                   ],
    }

    return '\n'.join(prefix + headers.get(context.arch, [])) + '\n'
```

Change it to, of course it is recommended, but for you want to run a AT&T project

```python
def _arch_header():
    prefix  = ['.section .shellcode,"awx"',
                '.global _start',
                '.global __start',
                '_start:',
                '__start:']
    headers = {
        'i386'  :  ['.p2align 0'],
        # 'x86'   :  ['.p2align 0'],
        'amd64' :  ['.intel_syntax noprefix', '.p2align 0'],
        'arm'   : ['.syntax unified',
                   '.arch armv7-a',
                   '.arm',
                   '.p2align 2'],
        'thumb' : ['.syntax unified',
                   '.arch armv7-a',
                   '.thumb',
                   '.p2align 2'
                   ],
        'mips'  : ['.set mips2',
                   '.set noreorder',
                   '.p2align 2'
                   ],
    }

    return '\n'.join(prefix + headers.get(context.arch, [])) + '\n'
```
