##Phase 2 Guide: Hybrid Teleoperation over Wi‑Fi and Ethernet

This guide walks you through setting up the SO101 follower arm on a
Raspberry Pi connected to your router via Ethernet, while controlling it
from your laptop over Wi‑Fi. Phase 2 introduces network variability due to
the laptop’s wireless connection, so additional steps are needed to ensure
reliable serial communication and teleop stability.

Prerequisites

Before starting, make sure you have:

SO101 follower arm connected to a Raspberry Pi (we used a Raspberry Pi 4
running Ubuntu 24.04 LTS). The follower arm’s USB cable should be
plugged into the Pi and enumerated as /dev/ttyACM0 or a similar device.

SO101 leader arm connected directly to your laptop over USB (the
leader shows up as /dev/ttyACM1 or another /dev/ttyACM* device).

Router with at least one free Ethernet port. The Pi is wired to this
router, while your laptop connects to the same router over Wi‑Fi.

Linux laptop (Ubuntu tested) with SSH and socat installed. The
laptop hosts the LeRobot teleop command.

SSH key pair for passwordless login from the laptop to the Pi. You
should have already added your laptop’s public key to
~ubuntu/.ssh/authorized_keys on the Pi.

1. Configure ser2net on the Raspberry Pi

The follower’s USB serial port must be shared over TCP so your laptop can
bridge into it. ser2net is a lightweight service that exposes serial
devices as network sockets.

Install ser2net if it is not already present:

sudo apt update
sudo apt install ser2net


Create or edit /etc/ser2net.yaml. Use the following minimal
configuration (adjust /dev/ttyACM0 and the baud rate if necessary):

%YAML 1.1
---
connection: &follower
  accepter: tcp,0.0.0.0,15000
  connector: serialdev,/dev/ttyACM0,1000000n81,local
  options:
    kickolduser: true
    max-connections: 1


accepter tells ser2net to listen on all interfaces (0.0.0.0) at
TCP port 15000.

connector points to the follower’s serial device and specifies a
baud rate of 1 Mb/s with 8 data bits, no parity, and one stop bit
(1000000n81). Set this to match your follower’s default baud rate
(commonly 115200 or 1000000).

kickolduser ensures only one TCP client can use the port at a time.

Restart the service and verify it is listening:

sudo systemctl restart ser2net
sudo systemctl status ser2net --no‑pager
sudo ss -lntp | grep ':15000'


You should see ser2net in a LISTEN state on port 15000. If the
status command shows errors (e.g. “Invalid accepter port name”), check
your YAML formatting—there must be no spaces after commas.

2. Set up SSH tunnelling on the laptop

Since the Pi is wired and the laptop is on Wi‑Fi, we create an SSH tunnel
from the laptop to the Pi to forward a local port (15001) to the Pi’s
ser2net port (15000). This ensures all serial data flows through a
secure connection.

ssh -N \
  -i ~/.ssh/id_ed25519 \   # path to your private key
  -o IdentitiesOnly=yes \
  -L 15001:<pi_ip>:15000 \
  ubuntu@<pi_ip>


Replace <pi_ip> with your Pi’s LAN IP (e.g. 192.168.10.86). The -N
flag tells SSH to create the tunnel without executing remote commands. This
process must remain running while you teleoperate; open it in its own
terminal.

Check that the tunnel is listening locally:

sudo ss -lntp | grep ':15001'


You should see a LISTEN entry owned by ssh. If the tunnel fails to
start, verify the Pi’s IP and that your SSH key is allowed.

3. Bridge the serial port with socat

On the laptop, run socat to create a pseudo‑terminal that forwards
traffic between /dev/ttyFOLLOWER and the SSH tunnel. socat acts as a
local serial device that LeRobot can open.

sudo socat -d -d \
  PTY,link=/dev/ttyFOLLOWER,raw,echo=0,mode=666 \
  TCP:127.0.0.1:15001,nodelay,keepalive


Explanation:

PTY,link=/dev/ttyFOLLOWER allocates a PTY and symlinks it at
/dev/ttyFOLLOWER so programs can consistently open the same path.

raw,echo=0 disables line buffering and echo; mode=666 sets
permissive permissions so non‑root processes can access it.

TCP:127.0.0.1:15001 connects to the local SSH tunnel. The
nodelay option reduces latency; keepalive helps detect dead
connections.

Keep socat running in its own terminal. To confirm it is working,
attempt to open the device:

ls -l /dev/ttyFOLLOWER
python3 - <<'PY'
import serial
s = serial.Serial('/dev/ttyFOLLOWER', 1000000, timeout=0.2)
print("Serial open OK")
s.close()
PY


If the serial open fails, ensure socat is still running and that the
SSH tunnel is connected.

4. Run teleoperation via LeRobot

With ser2net, SSH tunnelling, and socat in place, you can run
lerobot-teleoperate on your laptop. Use the pseudo‑terminal for the
follower and the real USB port for the leader. Example:

lerobot-teleoperate \
  --fps 10 \
  --robot.type=so101_follower \
  --robot.port=/dev/ttyFOLLOWER \
  --robot.id=follower \
  --teleop.type=so101_leader \
  --teleop.port=/dev/ttyACM1 \
  --teleop.id=leader


Key flags:

--fps 10 reduces the control loop rate to 10 Hz, easing pressure on
networked serial reads.

--robot.port=/dev/ttyFOLLOWER uses the socat PTY.

--teleop.port=/dev/ttyACM1 points to the leader’s USB serial port;
adjust this if your leader appears as a different device.

The teleop loop will run and print “Teleop loop time…” messages. If it
aborts with “There is no status packet!”, proceed to the next section.

5. Improve reliability with LeRobot patches

Wireless networks can introduce jitter and dropped packets. In Phase 2 we
found that LeRobot’s default sync_read aborts after a single missed
status packet. To handle temporary stalls, update the retry logic:

Locate lerobot/src/lerobot/motors/motors_bus.py and find the
_sync_read and sync_read functions.

Change the default retry count by setting num_retry: int = 5 in
both functions. This allows up to six attempts (initial + five
retries).

Add a short backoff between retries to avoid immediate repeated
failures:

# inside the for-loop after a failed txRxPacket
import time
time.sleep(0.01)


Reduce the teleop loop rate with --fps 10 as shown above.

For a ready‑to‑paste implementation and discussion of the patch, see
docs/patch_notes.md.

6. Troubleshooting

SSH tunnel fails: Make sure your laptop’s SSH key is present in
~ubuntu/.ssh/authorized_keys on the Pi. Confirm your Pi’s IP.

socat exits immediately: If socat logs “socket … is at EOF”,
verify that the SSH tunnel is still running and that ser2net is
listening on port 15000. Check sudo ss -lntp on the Pi.

Teleop aborts after some time: Apply the retry/backoff patch and
lower the frame rate as described above.

Different serial device names: If your follower enumerates as
/dev/ttyACM1 or another device, update both ser2net.yaml and the
lerobot-teleoperate command accordingly.

Next steps

After you achieve stable control in Phase 2, you can proceed to Phase 3:
disconnect the Pi from Ethernet, connect it to the router over Wi‑Fi, and
follow similar steps to bridge the serial port. Additional tuning may be
required to handle higher jitter on the Pi’s wireless link.

Good luck and happy teleoperating!
