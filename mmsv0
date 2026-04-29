#!/usr/bin/env python3
"""
4/29/2026 - Former File Name: micromouse_controlsunedited
4/29/2026 - Current File Name: mmsv0 
micromouse_controls.py
Micromouse Integrated Control — Reliable Sensor Initialization (merged mapping)
-------------------------------------------------------------------------------
- Runs per-sensor mapping (single-enable scan)
- Sequentially addresses three VL53L0X sensors (0x30,0x31,0x32) with verification
- Initializes IMU and magnetometer
- Motor control for TB6612FNG (time-based motion)
- Continuous autonomous obstacle avoidance loop
"""

import sys
import time
import board, digitalio
from gpiozero import PWMOutputDevice, DigitalOutputDevice
import adafruit_extended_bus
import adafruit_vl53l0x
import adafruit_lsm6ds.lsm6dsox
import adafruit_lis3mdl
import RPi.GPIO as GPIO

# =====================================================
# 1. I2C BUS SETUP  (/dev/i2c-3 overlay)
# =====================================================
i2c = adafruit_extended_bus.ExtendedI2C(3)

# =====================================================
# 2. MOTOR DRIVER SETUP (TB6612FNG)
# =====================================================
PWMA_PIN, PWMB_PIN = 12, 13
AIN1_PIN, AIN2_PIN = 15, 14
BIN1_PIN, BIN2_PIN = 16, 25
STBY_PIN = 18

pwmA = PWMOutputDevice(PWMA_PIN, frequency=1000, initial_value=0.0)
pwmB = PWMOutputDevice(PWMB_PIN, frequency=1000, initial_value=0.0)

AIN1 = DigitalOutputDevice(AIN1_PIN, initial_value=False)
AIN2 = DigitalOutputDevice(AIN2_PIN, initial_value=False)
BIN1 = DigitalOutputDevice(BIN1_PIN, initial_value=False)
BIN2 = DigitalOutputDevice(BIN2_PIN, initial_value=False)

STBY = DigitalOutputDevice(STBY_PIN, initial_value=False)

def motor_enable():
    STBY.on()

def motor_disable():
    STBY.off()

def set_motor_direction(l, r):
    """
    l, r: "F" (forward) or "R" (reverse) or "S" (stop/coast)
    TB6612 truth table:
      Forward: IN1=1 IN2=0
      Reverse: IN1=0 IN2=1
      Coast:   IN1=0 IN2=0
      Brake:   IN1=1 IN2=1
    """
    # Left motor (A channel)
    if l == "F":
        AIN1.on();  AIN2.off()
    elif l == "R":
        AIN1.off(); AIN2.on()
    else:  # "S"
        AIN1.off(); AIN2.off()

    # Right motor (B channel)
    if r == "F":
        BIN1.on();  BIN2.off()
    elif r == "R":
        BIN1.off(); BIN2.on()
    else:  # "S"
        BIN1.off(); BIN2.off()

def set_motor_power(lp, rp):
    """
    lp, rp are duty cycle percentages 0–100.
    gpiozero PWMOutputDevice.value expects 0.0–1.0.
    """
    lp = max(0.0, min(100.0, float(lp))) / 100.0
    rp = max(0.0, min(100.0, float(rp))) / 100.0
    pwmA.value = lp
    pwmB.value = rp

def stop_motors():
    set_motor_power(0, 0)
    set_motor_direction("S", "S")

def turn_left(t, p):
    # left motor reverse, right motor forward (spin left)
    set_motor_direction("R", "F")
    set_motor_power(p, p)
    time.sleep(t)
    

def turn_right(t, p):
    # left motor forward, right motor reverse (spin right)
    set_motor_direction("F", "R")
    set_motor_power(p, p)
    time.sleep(t)
    

def forward(t, p): #p = 25
    set_motor_direction("F", "F")
    set_motor_power(p, p)
    time.sleep(t)
    

def brake():
    # brake mode: IN1=IN2=1 for both channels
    AIN1.on(); AIN2.on()
    BIN1.on(); BIN2.on()
    set_motor_power(0, 0)

# =====================================================
# 3. VL53L0X mapping + reliable sequential init
# =====================================================

# XSHUT pin mapping for sensors (BCM)
# =====================================================
# 3. VL53L0X mapping + reliable sequential init
# =====================================================

# XSHUT pin mapping for sensors (BCM)
xshut_pins = [board.D17, board.D22, board.D27]  # right, front, left
target_addresses = [0x30, 0x31, 0x32]           # right, front, left

def scan_hex():
    return [hex(x) for x in i2c.scan()]

# 

import time
def log (level: str, msg: str)-> None : 
    print(f"{time.monotonic():10.3f} [{level}] {msg}", flush=True)
log("INFO", f"XHUST pins; L={xshut_pins}, F= {xshut_pins}, R= {xshut_pins}")
log("INFO", f"Target addrs: {[ hex(a) for a in target_addresses]} ")

#
 
def map_and_initialize_vl53(retries_per_sensor=5, boot_delay=0.50, address_commit_delay=0.10):
    xshuts = []
    sensors = []

    # Prepare XSHUTs and hold all sensors in reset (LOW)
    for pin in xshut_pins:
        p = digitalio.DigitalInOut(pin)
        p.direction = digitalio.Direction.OUTPUT
        p.value = False
        xshuts.append(p)
    time.sleep(0.10)

    # Optional: mapping (single-enable scan)
    print("\n=== Per-sensor single-enable scan ===\n")
    for i, pin in enumerate(xshut_pins):
        for p in xshuts:
            p.value = False
        time.sleep(0.02)
        xshuts[i].value = True
        time.sleep(boot_delay)
        print(f"S{i+1} XSHUT={pin} bus:", scan_hex())

    # Sequential addressing with retries and verification
    print("\n=== Sequential addressing (FIXED) ===\n")
    for i, addr in enumerate(target_addresses):
        print(f"\nAddressing S{i+1} -> {hex(addr)}")

        # Keep previously addressed sensors ON (0..i), keep future sensors OFF (i+1..)
        # This prevents already-addressed sensors from resetting back to 0x29.
        for j, p in enumerate(xshuts):
            p.value = (j <= i)
        time.sleep(boot_delay)

        success = False
        for attempt in range(1, retries_per_sensor + 1):
            before = i2c.scan()
            print(f"[S{i+1} try {attempt}] before: {[hex(d) for d in before]}")

            # At this moment, ONLY the current sensor should be at 0x29
            if 0x29 not in before:
                print("⚠ 0x29 not present; toggling current sensor and retrying")
                xshuts[i].value = False
                time.sleep(0.05)
                xshuts[i].value = True
                time.sleep(boot_delay)
                continue

            try:
                # Create driver at default address (0x29)
                vl = adafruit_vl53l0x.VL53L0X(i2c)
                time.sleep(0.02)

                # Change its address
                vl.set_address(addr)
                time.sleep(address_commit_delay)

                after = i2c.scan()
                print(f"           after: {[hex(d) for d in after]}")

                if addr in after:
                    sensors.append(vl)
                    print(f"✅ S{i+1} now at {hex(addr)}")
                    success = True
                    break
                else:
                    print("⚠ Address not visible after set_address; retrying…")

            except Exception as e:
                print(f"⚠ Exception while addressing S{i+1}: {e}")

            # Retry: toggle ONLY this sensor (do NOT toggle earlier sensors)
            xshuts[i].value = False
            time.sleep(0.05)
            xshuts[i].value = True
            time.sleep(boot_delay)

        if not success:
            print(f"🚫 Sensor S{i+1} failed after {retries_per_sensor} attempts")
            return None

    # Turn all sensors ON (do NOT reset them)
    for p in xshuts:
        p.value = True
    time.sleep(0.10)

    print("\nFinal bus:", scan_hex())

    # Optional: verify all target addresses present
    final = set(i2c.scan())
    for addr in target_addresses:
        if addr not in final:
            print(f"❌ Missing {hex(addr)} on final scan — sensor reset/glitched")
            return None

    return sensors

# =====================================================
# 4. IMU + MAGNETOMETER initialization (wrapped)
# =====================================================
def init_imu_and_mag():
    imu = None
    mag = None
    try:
        imu = adafruit_lsm6ds.lsm6dsox.LSM6DSOX(i2c, address=0x6B)
        print("✅ LSM6DSOX IMU initialized at 0x6B")
    except Exception as e:
        print(f"❌ IMU init failed: {e}")
        imu = None
    try:
        mag = adafruit_lis3mdl.LIS3MDL(i2c, address=0x1E)
        print("✅ LIS3MDL magnetometer initialized at 0x1E")
    except Exception as e:
        print(f"❌ Magnetometer init failed: {e}")
        mag = None
    return imu, mag

# =====================================================
# 5. MAIN program flow
# =====================================================
def main():
    # Initialize sensors (mapping + addressing)
    vl53_sensors = map_and_initialize_vl53()
    print('Sensors initializing... ') # <-- 
    if vl53_sensors is None:
        print("Aborting due to incomplete VL53 initialization.")
        safe_shutdown()
        sys.exit(1)

    # IMU + magnetometer
    imu, mag = init_imu_and_mag()

    # Enable motors
    motor_enable()
    print("\n🚀 Starting continuous autonomous loop (Ctrl+C to stop)…\n")

    try:
        count = 0
        while True:
            count += 1
                # --- ToF distances ---
            distances = []
            for s in vl53_sensors:
                try:
                    distances.append(s.range)
                except Exception:
                    distances.append(None)

            # --- IMU readings ---
            if imu:
                ax, ay, az = imu.acceleration
                gx, gy, gz = imu.gyro
            else:
                ax = ay = az = gx = gy = gz = None

            # --- Magnetometer readings ---
            if mag:
                mx, my, mz = mag.magnetic
            else:
                mx = my = mz = None

            # --- Print formatted output ---
            print(
                f"VL53L0X: {distances} mm | "
                f"Accel: ({ax:.2f}, {ay:.2f}, {az:.2f}) m/s² | "
                f"Gyro: ({gx:.2f}, {gy:.2f}, {gz:.2f}) rad/s | "
                f"Mag: ({mx:.2f}, {my:.2f}, {mz:.2f}) µT"
            )

            time.sleep(0.3)

            # read ToF distances
            distances = []
            for idx, s in enumerate(vl53_sensors):
                try:
                    d = s.range
                    distances.append(d)
                    print(f"ToF S{idx+1} (addr {hex(target_addresses[idx])}): {d} mm")
                except Exception as e:
                    distances.append(None)
                    print(f"⚠ Read error S{idx+1}: {e}")

            # check all
            if any(d is None for d in distances) or len(distances) < 3:
                print(f"[{count:04}] ⚠ Missing ToF reading → {distances}")
                time.sleep(0.2)
                continue

            right, front, left = distances
            print(f"[{count:04}] Right:{right}  Front:{front}  Left:{left}")

            # obstacle avoidance logic
            if front < 40:
                print("🚧 Wall ahead very close")
                brake()
               
                if left > right:
                    print("↩ Turning left")
                    turn_left(0.3, p=20)
                else:
                    print("↪ Turning right")
                    turn_right(0.3, p=20)

            elif left < 60 and right < 32 and front < 100:
                print("↩ Obstacle pattern detected: turning left")
                stop_motors()
                turn_left(0.6, p=20)

            elif front < 100 and left < 60:
                print("↪ Front and left blocked: turning right")
                stop_motors()
                turn_right(0.4, p=20)

            elif front < 100 and right < 32:
                print("↩ Front and right blocked: turning left")
                stop_motors()
                turn_left(0.4, p=20)
          

            else:
                print("⬆ Path clear: moving forward")
                forward(0.4, p=50)

            time.sleep(0.2)

    except KeyboardInterrupt:
        print("\n🛑 Stopped by user. Resetting sensors and cleaning up…")
        # drive all XSHUT low then high to free bus
        try:
            for pin in xshut_pins:
                p = digitalio.DigitalInOut(pin)
                p.direction = digitalio.Direction.OUTPUT
                p.value = False
            time.sleep(0.05)
            for pin in xshut_pins:
                p = digitalio.DigitalInOut(pin)
                p.direction = digitalio.Direction.OUTPUT
                p.value = True
        except Exception:
            pass
        brake()
        motor_disable()

    finally:
        safe_shutdown()
        print("✅ GPIO cleanup complete. System shut down safely.")

def safe_shutdown():
    """Stop motors, stop PWM (guarded), cleanup GPIO."""
    try:
        stop_motors()
    except Exception:
        pass
    try:
        pwmA.stop()
    except Exception:
        pass
    try:
        pwmB.stop()
    except Exception:
        pass
    try:
        GPIO.cleanup()
    except Exception:
        pass

if __name__ == "__main__":
    main()
