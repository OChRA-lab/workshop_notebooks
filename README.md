# Workshop guide

## Setup and Environments
Each PC has visual studio and anaconda installed, an environment has been setup called "sim_user"

Connect to the Wifi `Simulated_lab_network` with the password `simulation`

Lab IP for simulation is `192.168.137.1:8001`

If you want to explore the ui the url is `192.168.137.1:8001/app`
either register your own user or use the username: a and password: a


### Devices
Each Device will have its own set of methods you can call which will return an operation object you can use to interact with the data.

You have access to the following devices on these stations:

`chemspeed_station` -
- `chemspeed` - [ChemspeedSwing] 

`kuka_mobile_station` -
- `kuka2` - [KukaKMRiiwa]

`kuka_station` -

- `KUKA3` - [KukaKMRiiwa]

`lcms_station` -
- `lcms` - [WatersLCMS]

`quantos_station` -
- `quantos` - [MettlerQuantos]

`yumi_station` -
- `yumi` - [AbbYuMi]
- `camera` - [Webcam]
- `ika_plate` - [IkaPlate]
- `tecan` - [TecanXCalibur]
- `xpr` - [XprQuantos]

## Discovery classes

### Lab
The `Lab` class is the main entry point for connecting to the laboratory system. It provides access to stations and robots within the lab environment.

**Initialising**
Args:
    hostname (str): Ip address and port of the lab server.
    experiment_id (str, optional): ID of the experiment. Defaults to None.

**Methods**

- `get_station(station_name: str) -> Station`: Get a station object by name
- `get_stations() -> List[Station]`: Get all the connected stations  

**Example Usage:**
```python
from ochra_discovery.spaces.lab import Lab

# Connect to the lab
lab = Lab("lab_ip_here:8001", experiment_id="exp_001")

# Get a specific station
yumi_station = lab.get_station("yumi_station")

# Get all available stations
all_stations = lab.get_stations()

```

### Station
The `Station` class represents individual workstations within the lab. Each station can contain multiple devices and robots for specific experimental tasks.

**Methods**
- `get_device(device_identifier: Union[str, UUID]) -> Type[Device]`: Get a device object by name or UUID
- `get_robot(robot_identifier: Union[str, UUID]) -> Type[Device]`: Get a robot object by name or UUID

**Example Usage:**
```python
from ochra_discovery.spaces.station import Station
from uuid import UUID

# Get the station from the lab
station = lab.get_station("yumi_station")

# Get a device at this station
hotplate = station.get_device("ika_plate")
camera = station.get_device(UUID("device-uuid-here"))

# Get a robot at this station
yumi_robot = station.get_robot("yumi")
```

### Operations
The operations system tracks commands sent to devices and their results.

#### Operation
Represents a command given to a device with associated execution data.

```python
from ochra_discovery.equipment.operation import Operation
from uuid import UUID

# Initialize operation with database ID
operation_id = UUID("operation-uuid-from-database")
# or from a device operation
operation_object = camera.take_image()

operation = Operation(operation_id)

# Get the operation result object
result = operation.get_result_object()

# Get operation result data
data_bytes = operation.get_result_data()

# Save result data to file
operation.get_result_data(path="experiment_data")
```

#### OperationResult
Handles the results and data from completed operations.

```python
from ochra_discovery.equipment.operation_result import OperationResult
from uuid import UUID

# Initialize with result ID or use the operation method
result_id = UUID("result-uuid-from-operation")
result = OperationResult(result_id)
result = operation.get_result_object()

# Save result data with original filename
result.save_data()

# Save result data with custom filename
result.save_data("my_experiment_results")

# The system automatically handles different data types:
# - Single files are saved directly
# - Folders are automatically unzipped
```

### Storage
The storage system includes several classes for managing laboratory materials and containers.

#### Reagent
Represents chemical reagents with properties and amount tracking.

```python
from ochra_discovery.storage.reagent import Reagent

# Create a new reagent
water = Reagent(name="Distilled Water", amount=500.0, unit="mL")

# Add properties for documentation
water.add_property("purity", "99.9%")
water.add_property("supplier", "ChemCorp")
water.add_property("lot_number", "WH2O-2024-001")

# Modify reagent amount (e.g., after use)
water.change_amount(450.0)  # Used 50mL

# Remove a property if needed
water.remove_property("lot_number")
```

#### Vessel
Container objects that hold reagents with capacity management.

```python
from ochra_discovery.storage.vessel import Vessel
from ochra_discovery.storage.reagent import Reagent

# Create a vessel
beaker = Vessel(type="glass_beaker", max_capacity=250.0, capacity_unit="mL")

# Create reagents
nacl_solution = Reagent("NaCl Solution", 100.0, "mL")
buffer = Reagent("Phosphate Buffer", 50.0, "mL")

# Add reagents to the vessel
beaker.add_reagent(nacl_solution)
beaker.add_reagent(buffer)

# Remove a reagent if needed
beaker.remove_reagent(buffer)
```

#### Holder
Container that can hold other containers, useful for organizing multiple vessels.

```python
from ochra_discovery.storage.holder import Holder
from ochra_discovery.storage.vessel import Vessel

# Create a holder (e.g., test tube rack)
rack = Holder(type="test_tube_rack", max_capacity=24)

# Create vessels to hold
test_tube1 = Vessel("test_tube", 15.0, "mL")
test_tube2 = Vessel("test_tube", 15.0, "mL")

# Add containers to the holder
rack.add_container(test_tube1)
rack.add_container(test_tube2)

# Remove a container
rack.remove_container(test_tube1)
```

#### Inventory
Manages collections of containers and consumables for a specific owner.

```python
from ochra_discovery.storage.inventory import Inventory
from ochra_discovery.storage.vessel import Vessel

# Create an inventory for a user or station
user_inventory = Inventory(
    owner=station,  # or user object
    containers_max_capacity=100
)

# Create containers
sample_vial = Vessel("vial", 2.0, "mL")
storage_bottle = Vessel("bottle", 1000.0, "mL")

# Add containers to inventory
user_inventory.add_container(sample_vial)
user_inventory.add_container(storage_bottle)

# Remove container when no longer needed
user_inventory.remove_container(sample_vial)
```

## Manager Classes
### Lab

The `LabServer` class is the central server that manages the entire laboratory system. It coordinates stations, devices, robots, and operations through a REST API and web interface.

#### Setting up a lab

**Basic Lab Setup:**
```python
from ochra_manager.lab.lab_server import LabServer
from pathlib import Path

# Create the lab server
lab = LabServer(
    host="lab_ip_here",       # IP address to bind to
    port=8001,                    # Port for the server
    folderpath="./lab_data"       # Directory for data storage
)

# Start the server
lab.run() 

# This doesnt work on every system so you may have to do something like this instead
if __name__ == "__main__":
    uvicorn.run("my_lab:lab.app", host="0.0.0.0", port=8001, workers=8)
```

**Available Endpoints:**
- `/stations/*` - Station management (create, modify, call methods)
- `/devices/*` - Device control and monitoring
- `/robots/*` - Mobile robot coordination
- `/operations/*` - Operation tracking and results
- `/storage/*` - Laboratory inventory management
- `/app/*` - Web application interface

### Station

The `StationServer` class manages individual workstations within the laboratory. Each station runs independently and connects to the central lab server for coordination.

#### Station Methods

**Core Methods:**
- `setup(lab_ip)` - Initialize FastAPI app and connect to lab server
- `add_device(device)` - Register a device with the station
- `run()` - Start the station server
- `process_op(operation)` - Execute operations on devices or station
- `shutdown()` - Gracefully shutdown the station


#### Setting up a station

**Basic Station Setup:**
```python
from ochra_manager.station.station_server import StationServer
from ochra_common.spaces.location import Location
from ochra_common.utils.enum import StationType

# Define station location
location = Location(lab="ACL", room="main_lab", place="bench_1")

# Create the station server
station = StationServer(
    name="yumi_station",
    location=location,
    station_type=StationType.WORK_STATION,
    station_port=8007
)

# Setup and connect to lab server
station.setup(lab_ip="lab_ip_here:8001")

# Construct devices before adding them to the station
from abb_yumi.handler import AbbYuMi
from ika_rct_digital.handler import IkaPlate
from webcam.handler import Webcam
from tecan_xcalibur.handler import TecanXCalibur

# Create device instances with required parameters
yumi_tasks = [
    "pick_vial_from_rack",
]
yumi = AbbYuMi("yumi", yumi_tasks)
camera = Webcam(name="camera", usb_port="", index=0)


# Add the constructed devices to the station
station.add_device(yumi)
station.add_device(camera)

# Start the station server
station.run()
```

**Available Endpoints:**
- `/ping` - Health check
- `/` - Station overview
- `/devices` - List devices
- `/devices/{id}` - Device details
- `/devices/{id}/commands` - Execute device commands
- `/process_op` - Process operations from lab server
- `/hypermedia` - Interactive station controls
- `/shutdown` - Graceful shutdown



