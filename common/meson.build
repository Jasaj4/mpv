git = find_program('git', required: false)
if not git.found()
    git = ''
endif

generate_ver = '''
#!/usr/bin/env python3
import os
import sys
from subprocess import DEVNULL, CalledProcessError, check_output
if not sys.argv[1]:
    sys.exit(1)
git_cmd = [sys.argv[1], "--git-dir=" + os.path.join(sys.argv[2], ".git"),
           "--work-tree=" + sys.argv[2]]
describe_cmd = git_cmd + ["describe", "--abbrev=9", "--tags", "--dirty"]
try:
    ver = check_output(describe_cmd, stderr=DEVNULL, encoding="UTF-8")
except CalledProcessError:
    describe_cmd += ["--always"]
    ver = check_output(describe_cmd, stderr=DEVNULL, encoding="UTF-8")
    ver = f"v{sys.argv[3]}-dev-g{ver}"
# Fork marker (asssub: high-fps ASS subtitle rendering). SemVer build metadata.
sys.stdout.write(ver.strip() + "+asssub")
'''

version_h = vcs_tag(
    command: [python, '-c', generate_ver, git, source_root, meson.project_version().replace('-UNKNOWN', '')],
    fallback: 'v' + meson.project_version() + '+asssub',
    input: 'version.h.in',
    output: 'version.h',
    replace_string: '@VERSION@',
)

sources += version_h
