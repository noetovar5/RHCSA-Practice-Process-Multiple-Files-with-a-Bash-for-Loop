# RHCSA-Practice-Process-Multiple-Files-with-a-Bash-for-Loop
RHCSA Practice — Process Multiple Files with a Bash for Loop
Today’s focused objective: Use a Bash for loop to process multiple command-line arguments and generate a consolidated log report.
Estimated time: 45 minutes
Lab system: RHEL 9 or RHEL 10
Privileges required: None
Safety: All work stays under /tmp/rhcsa-loop-lab. This practice does not modify networking, storage, boot configuration, authentication, SELinux, firewall rules, or system services.
This builds directly on your previous Bash lesson. Red Hat’s current RHCSA objectives include using loops, processing script inputs, and processing command output in simple shell scripts. Official RHCSA EX200 objectives
1. The concept in plain language — 8 minutes
A loop repeats a set of commands.
Suppose three application logs must be checked. Without a loop, you might repeat the same command three times:
grep -c 'ERROR' inventory.log
grep -c 'ERROR' billing.log
grep -c 'ERROR' customer.log

A for loop lets you express the repeated operation once:
for file in inventory.log billing.log customer.log
do
    grep -c 'ERROR' "$file"
done

Bash assigns each item to the variable file, one at a time.
The standard structure
for variable in list
do
    commands
done

You may also place do on the first line:
for variable in list; do
    commands
done

Understanding "$@"
In a script, "$@" represents every supplied command-line argument while preserving each argument as a separate item.
If you run:
./check-all-logs.sh logs/app1.log logs/app2.log logs/app3.log

Then:
for log_file in "$@"

processes each pathname once.
Always prefer quoted "$@". An unquoted $@ can incorrectly split filenames containing spaces.
2. Prepare the safe lab — 5 minutes
Safety warning
The first command recursively deletes only an earlier copy of this temporary lab. Verify that the path is exactly /tmp/rhcsa-loop-lab.
rm -rf /tmp/rhcsa-loop-lab
mkdir -p /tmp/rhcsa-loop-lab/{logs,scripts,reports}
cd /tmp/rhcsa-loop-lab
pwd

Expected location:
/tmp/rhcsa-loop-lab

Create three application logs:
printf '%s\n' \
'2026-09-22 18:00:01 INFO InventoryApp started' \
'2026-09-22 18:01:10 WARN Database latency detected' \
'2026-09-22 18:02:15 ERROR Database connection failed' \
'2026-09-22 18:03:20 ERROR API returned HTTP 503' \
> logs/inventory.log

printf '%s\n' \
'2026-09-22 18:00:01 INFO BillingApp started' \
'2026-09-22 18:05:00 INFO Payment service connected' \
'2026-09-22 18:10:00 INFO Health check passed' \
> logs/billing.log

printf '%s\n' \
'2026-09-22 18:00:01 INFO CustomerApp started' \
'2026-09-22 18:03:00 WARN Authentication response was slow' \
'2026-09-22 18:04:30 ERROR Authentication provider unavailable' \
> logs/customer.log

Verify the starting files:
ls -l logs

Expected filenames:
billing.log
customer.log
inventory.log

Inspect them:
for file in logs/*.log; do
    echo "===== $file ====="
    cat "$file"
done

This is your first working loop:
- logs/*.log expands to matching log filenames.
- Each filename is assigned to file.
- cat "$file" displays the current file.
- Double quotes protect the pathname.
3. Guided practice: a basic counting loop — 6 minutes
Run:
for log_file in logs/*.log; do
    error_count=$(grep -c 'ERROR' "$log_file")
    echo "$log_file contains $error_count error entries"
done

Expected output:
logs/billing.log contains 0 error entries
logs/customer.log contains 1 error entries
logs/inventory.log contains 2 error entries

What happens during each pass:
1. Bash assigns one filename to log_file.
2. grep -c counts matching error lines.
3. $() captures that number.
4. echo displays the filename and its count.
5. Bash advances to the next file.
4. Guided practice: create a multiple-file script — 10 minutes
Create the script:
cat > scripts/check-all-logs.sh <<'EOF'
#!/bin/bash

if [[ $# -eq 0 ]]; then
    echo "Usage: $0 LOG_FILE [LOG_FILE ...]"
    exit 2
fi

for log_file in "$@"; do
    echo "Checking: $log_file"

    if [[ ! -f "$log_file" ]]; then
        echo "Result: FILE NOT FOUND"
        echo
        continue
    fi

    error_count=$(grep -c 'ERROR' "$log_file")
    warning_count=$(grep -c 'WARN' "$log_file")

    echo "Errors: $error_count"
    echo "Warnings: $warning_count"
    echo
done

exit 0
EOF

Make it executable and check its syntax:
chmod 750 scripts/check-all-logs.sh
bash -n scripts/check-all-logs.sh
echo "Syntax status: $?"

Expected:
Syntax status: 0

Important parts
if [[ $# -eq 0 ]]; then

This confirms that at least one argument was supplied.
for log_file in "$@"; do

This processes every supplied argument.
continue

continue stops the current loop iteration and moves to the next item. It prevents grep from trying to read a missing file.
Run the script:
./scripts/check-all-logs.sh \
  logs/inventory.log \
  logs/billing.log \
  logs/customer.log

Expected results include:
Checking: logs/inventory.log
Errors: 2
Warnings: 1

Checking: logs/billing.log
Errors: 0
Warnings: 0

Checking: logs/customer.log
Errors: 1
Warnings: 1

Test the missing-file logic:
./scripts/check-all-logs.sh \
  logs/inventory.log \
  logs/missing.log \
  logs/billing.log

The script should report the missing file and continue to billing.log.
Test the usage logic:
./scripts/check-all-logs.sh
echo "Exit status: $?"

Expected:
Usage: ./scripts/check-all-logs.sh LOG_FILE [LOG_FILE ...]
Exit status: 2

5. Guided practice: calculate totals — 8 minutes
Replace the script with this enhanced version:
cat > scripts/check-all-logs.sh <<'EOF'
#!/bin/bash

if [[ $# -eq 0 ]]; then
    echo "Usage: $0 LOG_FILE [LOG_FILE ...]"
    exit 2
fi

files_checked=0
missing_files=0
total_errors=0
total_warnings=0

for log_file in "$@"; do
    echo "Checking: $log_file"

    if [[ ! -f "$log_file" ]]; then
        echo "Result: FILE NOT FOUND"
        echo
        missing_files=$((missing_files + 1))
        continue
    fi

    error_count=$(grep -c 'ERROR' "$log_file")
    warning_count=$(grep -c 'WARN' "$log_file")

    echo "Errors: $error_count"
    echo "Warnings: $warning_count"
    echo

    files_checked=$((files_checked + 1))
    total_errors=$((total_errors + error_count))
    total_warnings=$((total_warnings + warning_count))
done

echo "===== SUMMARY ====="
echo "Files checked: $files_checked"
echo "Missing files: $missing_files"
echo "Total errors: $total_errors"
echo "Total warnings: $total_warnings"

if [[ $missing_files -gt 0 ]]; then
    exit 3
elif [[ $total_errors -gt 0 ]]; then
    exit 1
else
    exit 0
fi
EOF

Check the syntax again:
bash -n scripts/check-all-logs.sh
echo "Syntax status: $?"

Arithmetic expansion
This syntax performs integer arithmetic:
$((expression))

For example:
files_checked=$((files_checked + 1))

adds one to the existing value.
This line adds the current file’s errors to the running total:
total_errors=$((total_errors + error_count))

Run the complete report:
./scripts/check-all-logs.sh logs/*.log
report_status=$?
echo "Report exit status: $report_status"

Expected summary:
===== SUMMARY =====
Files checked: 3
Missing files: 0
Total errors: 3
Total warnings: 2
Report exit status: 1

The script returns 1 because errors were found. That is an intentional monitoring result, not a scripting failure.
Test missing-file priority:
./scripts/check-all-logs.sh logs/inventory.log logs/missing.log
report_status=$?
echo "Report exit status: $report_status"

Expected summary values include:
Files checked: 1
Missing files: 1
Total errors: 2
Total warnings: 1
Report exit status: 3

The final condition gives missing input a higher priority than detected application errors.
6. Save and verify a report — 3 minutes
Generate a report:
./scripts/check-all-logs.sh logs/*.log > reports/application-summary.txt
report_status=$?

Display it:
cat reports/application-summary.txt
echo "Saved exit status: $report_status"

Verify important totals:
grep -q '^Files checked: 3$' reports/application-summary.txt \
  && echo "PASS: three files checked"

grep -q '^Total errors: 3$' reports/application-summary.txt \
  && echo "PASS: three total errors"

grep -q '^Total warnings: 2$' reports/application-summary.txt \
  && echo "PASS: two total warnings"

The anchors ensure that each verification matches the complete expected line.
7. Independent challenge — 5 minutes
Create this script without copying the guided script:
scripts/check-permissions.sh

Requirements
The script must:
1. Accept one or more pathnames as arguments.
2. Display a usage message and exit 2 when no arguments are supplied.
3. Process each pathname using a for loop and "$@".
4. For each pathname:
   - Display its name.
   - Report MISSING if it does not exist.
   - Report EXECUTABLE if it exists and is executable.
   - Report NOT EXECUTABLE if it exists but is not executable.
5. Use continue when a pathname is missing.
6. Count:
   - Existing paths checked
   - Missing paths
   - Executable paths
7. Display the three totals after completing the loop.
8. Pass bash -n.
9. Be executable by its owner.
Test it with:
scripts/check-all-logs.sh
logs/inventory.log
logs/missing.log

Expected facts:
- check-all-logs.sh is executable.
- inventory.log exists but is not executable.
- missing.log does not exist.
- Existing paths checked: 2
- Missing paths: 1
- Executable paths: 1
8. Independent verification
Run:
bash -n scripts/check-permissions.sh \
  && echo "PASS: syntax is valid" \
  || echo "FAIL: syntax error"

Confirm it is executable:
test -x scripts/check-permissions.sh \
  && echo "PASS: challenge script is executable" \
  || echo "FAIL: challenge script is not executable"

Save its output:
./scripts/check-permissions.sh \
  scripts/check-all-logs.sh \
  logs/inventory.log \
  logs/missing.log \
  > reports/permission-summary.txt

Then verify the required totals:
grep -q '^Existing paths checked: 2$' reports/permission-summary.txt \
  && echo "PASS: existing-path count is correct"

grep -q '^Missing paths: 1$' reports/permission-summary.txt \
  && echo "PASS: missing-path count is correct"

grep -q '^Executable paths: 1$' reports/permission-summary.txt \
  && echo "PASS: executable-path count is correct"

These checks verify your results without revealing the challenge solution.
9. Knowledge check
Answer these without reviewing the lesson:
1. What is the purpose of a for loop?
2. What does "$@" represent?
3. Why is "$@" preferable to unquoted $@?
4. What do do and done mark?
5. What does continue do?
6. How do you increase count by one using Bash arithmetic?
7. What does logs/*.log mean?
8. When does the shell expand the * wildcard?
9. What is the difference between $# and "$@"?
10. Why should variables containing pathnames be quoted?
11. What does [[ -x "$path" ]] test?
12. What does bash -n verify?
13. Why must $? be saved immediately?
14. If three log files contain two, zero, and one errors, what should the total be?
I would consider this objective mastered when you can process an unknown number of inputs, validate each one, continue past invalid input, maintain totals, and return a meaningful result.
Safe cleanup
Safety warning
The following command recursively removes the complete temporary lab. Confirm that the path is exactly /tmp/rhcsa-loop-lab.
cd /tmp
rm -rf /tmp/rhcsa-loop-lab

Verify:
test ! -e /tmp/rhcsa-loop-lab \
  && echo "PASS: loop lab removed safely" \
  || echo "CHECK: loop lab still exists"

Expected:
PASS: loop lab removed safely





RHCSA Practice — Build a Log-Checking Bash Script Today’s focused objective: Create a simple Bash script that accepts a command-line argument, validates input with an if statement, processes command output, and returns a meaningful exit status. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: This lesson works only under /tmp/rhcsa-script-lab. It does not modify services, networking, storage, boot configuration, authentiSep 20RHCSA Practice — Search Logs with grep and Regular Expressions Today’s focused objective: Use grep, pipes, and regular expressions to locate useful information in configuration files and application logs. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneSafety: Everything stays under /tmp/rhcsa-grep-lab. This lesson does not change networking, storage, boot, authentication, SELinux, firewall rules, or system services. This isSep 17RHCSA Practice — Archive and Compress Files with tar Today’s focused objective: Create, inspect, extract, compress, and verify archives using tar, gzip, and bzip2. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10Privileges required: NoneExam relevance: Red Hat’s current EX200 objectives require candidates to archive, compress, unpack, and uncompress files using tar, gzip, and bzip2. [Official RHCSA EX200 objectives](https://www.redhat.com/en/Sep 15RHCSA Practice — Hard and Symbolic Links Today’s focused objective: Create, inspect, troubleshoot, and remove hard links and symbolic links. Estimated time: 45 minutesLab system: RHEL 9 or RHEL 10 VMExam relevance: Creating hard and soft links is listed under the current RHCSA EX200 essential-tools objectives. Official Red Hat EX200 objectives This lesson builds on yoSep 13RHCSA Practice — Standard Linux Permissions Today’s objective: Read, set, and verify standard user/group/other permissions using symbolic and numeric notation. Estimated time: 45 minutesLab system: Your RHEL 9 VM is suitable for this practice.Safety: This lab works entirely inside /tmp/rhcsa-permissions-lab. It does not change networking, storage, boot settings, authentication, SELinux, the firewall, or system availability. Red Hat’s current EX200 objectives includSep 10












    To pick up a draggable item, press the space bar.
    While dragging, use the arrow keys to move the item.
    Press space again to drop the item in its new position, or press escape to cancel.
