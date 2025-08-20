# UCSD RoboCar Project - Team 09
## ECE MAE 148 - Autonomous Vehicles

Reference: [UCSD ECE MAE 148 Course Website](https://ucsd-ecemae-148.github.io/)

---

## Part 1: Jetson Nano Connection & Initial Setup

### SSH Connection
```bash
# Direct USB connection
ssh jetson@192.168.55.1

# WiFi connection (after setup)
ssh jetson@ucsdrobocar-148-09.local

# Password
manchester
```

### WiFi Network Configuration

1. **Connect via USB first** to configure WiFi
2. **List available networks:**
   ```bash
   sudo nmcli device wifi list
   sudo nmcli device wifi rescan  # if needed
   ```

3. **Connect to UCSD RoboCar Network:**
   ```bash
   # 5GHz (preferred)
   sudo nmcli device wifi connect UCSDRoboCar5GHz password UCSDrobocars2018
   
   # 2.4GHz fallback
   sudo nmcli device wifi connect UCSDRoboCar password UCSDrobocars2018
   ```

4. **Get IP address:**
   ```bash
   ifconfig
   ```

5. **Disable WiFi power saving for stable connection:**
   ```bash
   sudo iw dev wlan0 set power_save off
   
   # Make persistent
   sudo nano /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
   # Change wifi.powersave = 3 to wifi.powersave = 2
   ```

### System Configuration
- **Hostname:** ucsdrobocar-148-09
- **Username:** jetson  
- **Password:** manchester
- **Date/Time:** Ensure NTP sync is enabled

---

## Part 2: Donkeycar Framework Setup

### D4 Project Structure
```
~/projects/
├── d4/                    # Main Donkeycar project
│   ├── manage.py         # Main control script
│   ├── myconfig.py       # Configuration file
│   ├── config.py         # Base configuration
│   ├── data/             # Training data (tubs)
│   │   └── tub_*/        # Individual recording sessions
│   └── models/           # Trained models
```

### Configuration (myconfig.py)
```python
# Camera
CAMERA_TYPE = "MOCK"  # Change to "WEBCAM" or "OAKD" for actual camera

# VESC Motor Controller
DRIVE_TRAIN_TYPE = "VESC"
VESC_SERIAL_PORT = "/dev/ttyACM0"
VESC_MAX_SPEED_PERCENT = 0.2
VESC_HAS_SENSOR = True
VESC_START_HEARTBEAT = True
VESC_BAUDRATE = 115200
VESC_TIMEOUT = 0.05
VESC_STEERING_SCALE = 0.5
VESC_STEERING_OFFSET = 0.5

# Controller
USE_JOYSTICK_AS_DEFAULT = True
CONTROLLER_TYPE = 'F710'
JOYSTICK_DEADZONE = 0.01
JOYSTICK_MAX_THROTTLE = 0.5
```

### Data Collection

1. **Start driving mode:**
   ```bash
   cd ~/projects/d4
   python manage.py drive
   ```
   
2. **Web interface:** Navigate to `http://ucsdrobocar-148-09:8887`

3. **Joystick controls:**
   - Start/Stop recording: Button A/X
   - Throttle: Right stick
   - Steering: Left stick
   - Emergency stop: Button B

4. **Data organization:**
   - Each session creates a new tub: `data/tub_<number>_<date>`
   - Clean bad data before training
   - Backup important sessions

### Training on GPU Cluster

#### Initial Setup
```bash
# SSH to cluster
ssh YOUR_UCSD_USERNAME@ieng6.ucsd.edu

# Prepare environment
prep me148f

# Create project structure (one-time setup)
launch-scipy-ml.sh -i ucsdets/donkeycar-notebook:latest
source activate /datasets/conda-envs/me148f-conda/
donkey createcar --path projects/d4/
exit
```

#### Data Transfer (rsync)

**From Jetson to GPU Cluster:**
```bash
# Transfer all data
rsync -avr -e ssh --rsync-path=cluster-rsync ~/projects/d4/data/* \
  YOUR_USERNAME@ieng6.ucsd.edu:projects/d4/data/

# Transfer specific tub
rsync -avr -e ssh --rsync-path=cluster-rsync ~/projects/d4/data/tub_1_25-01-15 \
  YOUR_USERNAME@ieng6.ucsd.edu:projects/d4/data/
```

#### Training Process

1. **Launch GPU container:**
   ```bash
   ssh YOUR_USERNAME@ieng6.ucsd.edu
   
   # For training (with GPU)
   launch-scipy-ml.sh -c 8 -m 32 -g 1 -i ucsdets/donkeycar-notebook:latest -v 2080ti
   
   # Navigate to project
   cd ~/projects/d4
   ```

2. **Train model:**
   ```bash
   # Train on all tubs
   python train.py --model=models/my_model.h5 --type=linear
   
   # Train on specific tubs
   python train.py --tub=data/tub_1,data/tub_2 --model=models/my_model.h5
   
   # Transfer learning
   python train.py --tub=data/new_tub --transfer=models/base_model.h5 \
     --model=models/improved_model.h5
   ```

3. **Exit and cleanup:**
   ```bash
   exit
   kubectl get pods  # Should show "No resources found"
   
   # If pods still running
   kubectl delete pod <pod_name>
   ```

#### Model Transfer Back to Jetson

```bash
# From local machine
rsync -avr -e ssh YOUR_USERNAME@ieng6.ucsd.edu:projects/d4/models/*.h5 \
  jetson@ucsdrobocar-148-09.local:~/projects/d4/models/
```

### Autonomous Driving

```bash
cd ~/projects/d4
python manage.py drive --model=models/my_model.h5

# Or use JSON model (loads faster)
python manage.py drive --model=models/my_model.json
```

### Continuous Training Workflow

**continuous_data_rsync.py** (on Jetson):
```python
import os
import time

while True:
    command = "rsync -aW --progress ~/projects/d4/data/ " \
              "YOUR_USERNAME@ieng6.ucsd.edu:projects/d4/data/ --delete"
    os.system(command)
    time.sleep(10)
```

Run with: `python continuous_data_rsync.py`

---

## Part 3: GPS Integration

### Setup
```bash
# Bash alias already configured
alias start_gps='runner'
```

### Usage
```bash
# Start GPS runner in new terminal
start_gps

# GPS data will be available at configured port
# Check /dev/ttyACM* or /dev/ttyUSB* for GPS device
```

### Integration with Donkeycar
- GPS data logging integrated with tub recording
- Coordinates saved with each frame
- Used for lap timing and position tracking

---

## Troubleshooting

### Common Issues

1. **VESC not detected:**
   ```bash
   ls /dev/ttyACM*
   # Reconnect USB if not showing
   ```

2. **Permission errors:**
   ```bash
   sudo usermod -a -G dialout jetson
   # Logout and login again
   ```

3. **Import errors (pyvesc):**
   ```bash
   pip install git+https://github.com/LiamBindle/PyVESC.git
   ```

4. **Camera issues:**
   - Start with `CAMERA_TYPE = "MOCK"` for testing
   - Ensure camera cables are properly connected
   - Check `v4l2-ctl --list-devices` for USB cameras

### Performance Tips
- Keep models under 10MB for faster loading
- Use JSON format for production models
- Monitor temperature: `watch -n 1 tegrastats`
- Ensure adequate cooling (fan should be running)

---

## Important Notes

- **Always exit GPU containers** when done to free resources
- **Backup data regularly** - use multiple tub directories
- **Test incrementally** - verify each component before integration
- **Document changes** to myconfig.py for reproducibility

---

## Next Steps
- [ ] Complete GPS integration testing
- [ ] Implement lap timer functionality
- [ ] Add telemetry logging
- [ ] Optimize model for real-time performance

---

*Last Updated: Current Session*  
*Team 09 - ECE MAE 148*
