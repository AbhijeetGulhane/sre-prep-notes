# Drill log - exit tests

Rules:
- Write the exit test BEFORE attempting. It must be observable (a command output, a state, an explanation given aloud).
- PASS requires: criterion met, within the time limit, Cold = Y (no notes, docs, or AI).
- FAIL gets a named gap and a retest date. A drill is closed only after a cold PASS.
- Lab box: rhel-lab (RHEL 9.8). Reset: virsh destroy rhel-lab; virsh snapshot-revert rhel-lab clean; virsh start rhel-lab

| Date | Drill | Area | Exit test | Limit | Taken | Cold? | Result | Gaps | Retest |
|---|---|---|---|---|---|---|---|---|---|
