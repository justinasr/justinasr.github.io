# Writing python subprocess.Popen stdout and stderr to file

`subprocess.Popen` function accepts two arguments - `stdout` and `stderr` where output and error streams will be written. It is possible to give file objects and write output straight to files. Dummy script that prints output to both output and error streams:

```python
import sys

print('stdout 1')
print('stderr 1', file=sys.stderr)
print('stdout 2')
print('stderr 2', file=sys.stderr)
print('stdout 3')
print('stderr 3', file=sys.stderr)
```

Script that runs the dummy script and points stdout and stderr streams to files:

```python
from subprocess import Popen as popen


cmd = 'python3 test.py'

# stdout and stderr to console
print('stdout and stderr to console')
proc = popen(cmd, shell=True)
print(proc.communicate())

# stdout to file and stderr to console
print('stdout to file and stderr to console')
with open('stdout.log', 'w') as log_file:
    proc = popen(cmd, shell=True, stdout=log_file)
    print(proc.communicate())

# stdout to console and stderr to file
print('stdout to console and stderr to file')
with open('stderr.log', 'w') as log_file:
    proc = popen(cmd, shell=True, stderr=log_file)
    print(proc.communicate())

# stdout and stderr to file
print('stdout and stderr to file')
with open('stdout_stderr.log', 'w') as log_file:
    proc = popen(cmd, shell=True, stdout=log_file, stderr=log_file)
    print(proc.communicate())
```
