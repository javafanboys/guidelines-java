# Small Java scripts and one-off coding tasks

Read before writing a one-off script or small coding task.

- In windows claude-code for any one time script and small coding tasks, most of the time powershell works.
- Otherwise prefer single file java for small coding tasks such as project utility/testing/automating ci/cd/git/build workflow. These serve as powerful replacement for bash/python/batch/ps1 scripts which might turn fragile especially in cross platform scenarios. More so our teams are more comfortable in java, python we don't know, and we find it easy to maintain java, and also IDE code completion and all of that works out of the box.
- Opening/shebang of such files should be `///usr/bin/env java "$0" "$@" ; exit $?` if no dependencies required or `///usr/bin/env jbang "$0" "$@" ; exit $?` if it uses maven dependencies or needs more capabilities thus needing jbang.
- This declaration should immediately follow with this single line comment which sets an informative context, `// Standalone Java code (not part of main project) - replaces bash/python/batch scripts with IDE-friendly, maintainable code using JDK 11/21/25 enhancements. To know why, refer to Cay Horstmann's JavaOne 2025 talk "Java for Small Coding Tasks" (https://youtu.be/04wFgshWMdA)`
- And then we can continue as usual.
- File names of such java source files should not have dashes (`-`) unlike how this is allowed in bash/batch file names; rather even single/simple java source files that are intended for simple coding tasks - and not part of main project should be named properly as per standard camel-case conventions.
- These files can be anywhere, they are NOT part of the project code and will not follow any convention specific to the project (whether java maven project, or anything else), and so these can be saved in preferably `test-scripts` folder.