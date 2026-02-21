# so101_teleop_hybrid
Welcome to SO101 Teleop Phase 2, a standalone guide and configuration set for
controlling the SO101 follower arm over a hybrid network. In this phase the
follower arm remains wired to the router via Ethernet, while the leader arm
and teleop laptop operate over Wi‑Fi. The goal is to achieve reliable
teleoperation despite the added network variability by carefully configuring
serial–to‑network bridging, port forwarding, and LeRobot’s control loop.

This repository contains:

docs/phase2-guide.md – a detailed, step‑by‑step tutorial that covers
Raspberry Pi setup, ser2net configuration, SSH tunnelling, socat port
forwarding, running the teleop command with proper flags, and tuning for
network stability.

docs/patch_notes.md – a changelog describing the code modifications and
runtime tweaks applied to LeRobot to make teleoperation robust over Wi‑Fi.

configs/ser2net.yaml – a minimal configuration file for ser2net that
exposes your follower’s USB serial device on a TCP port.

You can clone or download this repository, copy the configuration to your
Pi, and follow the guide to perform Phase 2 teleoperation. Since the
GitHub connector used in this project is read‑only, creating a new
repository or adding files must be done manually through the GitHub web
interface. See docs/phase2-guide.md for detailed instructions.

Repository structure
so101_teleop_phase2/
├── README.md               – This overview
├── configs/
│   └── ser2net.yaml        – Example ser2net configuration
└── docs/
    ├── phase2-guide.md     – Step‑by‑step phase 2 tutorial
    └── patch_notes.md      – Notes on code tweaks and retries


© 2026 SO101 Teleop Project. Feel free to modify and adapt these files to
match your own hardware (device names, baud rates, etc.) and to document
additional improvements.
