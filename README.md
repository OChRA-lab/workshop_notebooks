# Terminology

Key terms used in Ochra:

- **Device**: The physical instrument or robot you want to control (e.g., a robot, hotplate, or other scientific equipment).
- **Driver**: The software that controls the device.
- **Handler**: OChRA's wrapper for a driver, exposing its capabilities to the framework.
- **OChRA Device**: A class that lets users interact with the device through OChRA.
- **Station**: A RESTful server representing a physical location in the lab. It hosts connected devices and processes incoming requests via handlers.
- **Lab**: A RESTful server managing multiple stations. It handles routing, scheduling, and database interactions.
- **Operation**: Represents an request within OChRA.

# Package Overview
OChRA is a lab integration and automation framework designed to make it easy to control and coordinate diverse scientific equipment. It's made up of three core repositories:

- **ochra_common**: Shared utilities and base classes used across all OChRA components.
- **ochra_manager**: Tools for building and managing labs and stations. Intended for automation engineers.
- **ochra_discovery**: Interfaces for designing and executing workflows. Intended for scientists and experiment designers.

## Typical Usage 
Install `ochra_common`, then add either 
- `ochra_manager` for setting up labs and stations, or
- `ochra_discovery` for creating and running workflows.

## What OChRA is not
OChRA doesn't replace device-specific control code. It lets you connect and coordinate many devices from one place.
In the provided example, we use a driver package called [pylabware](https://github.com/croningp/pylabware), which handles interactions with equipment for a few of our devices.
You will need to write your own drivers for any device that don't have one using any APIs or SDKs provided by the manufacturer.
We have written a few drivers and can probably help you figure out what you need. Ultimately, all you need for your equipment to be usable in OChRA is to have methods that can be invoked by the handler class.

## How Does It Work

In the simplest case, we would have a lab comprised of a single station that has one piece of equipment attached to it. For this, we would have to run two servers: 
- A **lab server** for coordination and data
- A **station server** for device control

These servers can be running on the same machine or on different machines connected to the same network.

### Step-by-step Execution
1. **Start Lab Server**:  Launch the lab script to start the lab server and initialize the database.
2. **Start Station Server**: Launch the station script to start the station server. It initializes handlers and registers with the lab.
3. **Running a Workflow**: When a user starts a workflow (e.g., a workflow with a single command to start stirring), the process follows this flow:
    1. **Connect to the Lab:** The user first establishes a connection with the lab.
    2. **Invoke a Method:** Using the front-end OChRA device class, the user invokes a method (e.g., `start_stir`). This is translated into an HTTP request sent to the lab.
    3. **Construct an Operation:** The lab constructs an operation and logs it to the database. Next, it determines which station the operation needs to be sent to.
    4. **Schedule Operation** the operation is passed to the scheduler, which queues operations till the appropriate station is available. Currently, a First In, First Out (FIFO) queue is employed.
    5. **Forward to Station:** The operation is passed to the appropriate station, which executes the desired method using the handler class.
    6. **Return Results:** Any results produced by the method invocation are stored as an OperationResult instance and assigned to the original Operation instance. The operation is passed back to the user as a reference.

    This workflow allows for seamless communication and control of equipment through OChRA's framework.

# Installation and Usage

## Installing

First, we need to install [mongoDB](https://www.mongodb.com/docs/manual/installation/)

Next, install `ochra_common` package. 
Using your favorite environment manager, set up your workspace and clone `ochra_common` repository into your desired location using the following command
```
git clone https://github.com/OChRA-lab/ochra_common.git
```
then using your environment install the package
```
pip install ./ochra_common
```

Then, depending on your goal, i.e., setting up lab/station or programming a workflow, repeat these steps to install either `ochra_manager` or `ochra_discovery` (or both if you want to test everything in one place).

To control a piece of equipment, check our organization page for the equipment package and install it as well.  

*Note: We plan to offer modular installation options soon. For now, please manually ensure all required packages are installed.*

## Setup

You can skip this section if you are only interested in running a workflow and someone else has already set up your labs, stations, and equipment.  
We'll use the IKA plate as an example.

### Equipment setup
You need to make a package for your equipment and install it so it can be utilised by both ochra_discovery and ochra_manager users.

Our equipment packages are structured to be comprised of three files: 
- `abstract.py` contains shared definitions and data model.
- `handler.py` exposes device functionality through OChRA. Calls the driver of the device.
- `device.py` contains the front end methods called for execution

You will need to create these files for every piece of equipment you want to incorporate in OChRA.

We have a script in the `ochra_devices` package that sets up the package structure, but it's a bit out of date at the moment.

#### Abstract

This is where you define the common data structure and shared methods for your device in both the device and handler classes. The only requirement is to inherit from the `Device` class in `ochra_common`.

This abstract class can be as comprehensive as you like but its ultimately just a data structure and template. For our example IKA plate, its abstract file:

```python
from ochra_common.equipment.device import Device
from pydantic import Field

class IkaPlateAbstract(Device):
    temperature: float = Field(default=0)
    stir_speed: float = Field(default=0)
```

#### Handler

This class represents an OChRA wrapper for the device's driver so its functionalities can be exposed in the framework.

First, import and inherit from the `abstract` class defined as previously described and `RestProxyMixin` class from `ochra_common`.  
Define `__init__` method; the first call should always be the superclass init with the `name` field passed through.  
The module path will be `"YOUR_MODULE.device"` and collection `"devices"`.

```python
from .abstract import IkaPlateAbstract
from ochra_common.utils.mixins import RestProxyMixin

class IkaPlate(IkaPlateAbstract, RestProxyMixin):
```

Next, do any necessary initialisation for your driver in order for its methods to be callable inside the OChRA handler.  

For the IKA plate driver, we use `pylabware` package, which provides drivers for various common lab instruments. Using that driver, we carry out the needed initialisation to construct it and connect to the real instrument.  
Any properties you don't want exposed by the OChRA handler (such as the driver) should be prefixed with an underscore (`_`) to be ignored.

Add the mixin hook. The `_endpoint` and `id` properties are created by the base device class and generated on creation, so just pass these to the mixin hook. This allows any property changes to be mapped to the device in Ochra's DB.

```python
    def __init__(self, name, connection_mode, port) -> None:
        super().__init__(name=name, module_path="ika_rct_digital.device")
        self._driver = RCTDigitalHotplate(name, connection_mode, "", port)
        self._driver.connect()
        self._mixin_hook(self._endpoint, self.id)
```

Now, add the methods you want to expose to the user. In this example, we add `start_stir`, `stop_stir`, and `set_speed`.

```python
    def start_stir(self) -> bool:
        return self._driver.start_stirring()

    def stop_stir(self) -> bool:
        return self._driver.stop_stirring()

    def set_speed(self, speed: int) -> bool:
        self.stir_speed = speed
        return self._driver.set_speed(speed=speed)
```

That's it for the handler. You can define as many methods as you like here, and they can be as complex or simple as you want. The only requirement is that they have a corresponding 'caller' in the device class.

##### TODO UI stuff here

#### Device

This class provides an interface for the device so that any OChRA user can operate the device via the `ochra_discovery` package when developing a workflow.

Similar to the handler class, import and inherit from the `abstract` class but this time inherit from `RestProxyMixinReadOnly` instead.  
You are required to provide the `name` as part of the init method, as this will be used to find the corresponding device handler when executing issued commands.  
No other things are required for initialization, but if you want to add a logger or something similar, you can do so between the super call and the mixin hook. For the IKA example:

```python
from .abstract import IkaPlateAbstract
from ochra_common.utils.mixins import RestProxyMixinReadOnly

class IkaPlate(IkaPlateAbstract, RestProxyMixinReadOnly):
    def __init__(self, name):
        super().__init__()
		# insert logger here
        self._mixin_hook(self._endpoint, name)
```

Now, add the methods. In the future, these will be auto-generated, but for now, add a method for each handler method you want to expose to the user.  
Use the `_lab_conn` property and the `call_on_object` function, which takes four arguments: *an endpoint, an ID, a function to call, and a dictionary containing the arguments for that function*. In the context of our IKA example, this looks like the following:

```python
    def set_speed(self, speed: int) -> bool:
        return self._lab_conn.call_on_object(self._endpoint, self.id,
                                             "set_speed",
                                             {"speed": speed})

    def start_stir(self) -> bool:
        return self._lab_conn.call_on_object(
            self._endpoint, self.id, "start_stir", {})

    def stop_stir(self) -> bool:
        return self._lab_conn.call_on_object(
            self._endpoint, self.id, "stop_stir", {})
```

### Lab setup

In OChRA framework, each lab would have a single lab server that handles the lab's data and manages its stations and their operation execution.

Setting up a lab is quick and easy. Use a short script similar to [my_lab.py](https://github.com/OChRA-lab/ochra_examples/blob/main/my_lab.py) to initialize your lab server and then run it.  
You can customize the server's hostname and port with `uvicorn`. For debugging and testing, we also delete the OChRA DB on startup, but you can skip this if you don't need to. See below:

```python
from ochra_manager.connections.db_connection import DbConnection
from ochra_manager.lab.lab_server import LabServer
import uvicorn

db = DbConnection()
# Clear the DB if it exists
db.db_adapter.delete_database()

# Construct lab server and start running
lab = LabServer("0.0.0.0", 8001)

if __name__ == "__main__":
    uvicorn.run("my_script:lab.app", host="0.0.0.0", port=8001, workers=8)
```

### Station setup
In OChRA framework, a station represents a physical location in the lab that have a number of instruments and robots attached to it that users can operate to run their workflows.

To start a station, use the `StationServer` class, provide the station's name, location and its type. For example: 

```python
my_station = StationServer(
    name="ika_station",
    location=Location(lab="my_lab", room="main_lab", place="Back left side of the room"),
    station_type=StationType.WORK_STATION,
)
```

Connect the station to the lab server by calling the `setup` method and passing the lab IP address and port number (set to 8001 in the example above)

```python
my_station.setup(lab_ip="my_lab_ip:8001")
```

Next, set up your device handler. Make sure you install the device package as previously instructed. Use the station's `add_device` method to add the IKA plate to the station's list of connected devices. Note that a station can have any number of devices so you can add any more handlers that you need to create your station but for this example we are only adding one.

```python
# Construct device
ika = IkaPlate(name="my_ika")

# Add device to the station server
my_station.add_device(ika)
```

Run the station with the `run` method.

```python
my_station.run()
```

### Running your setup
Now you should have 2 scripts, one for your lab and one for your station, so now we need to run them both. In our setup, we have these running on two separate machines so that our lab server is running on a central machine where data can be securely stored, while our station server is running remotely in the lab.

Firstly, you should run the lab script, this will setup your database and you should be able to access it at the address "YourSetUpIP:8001/docs" and you will see an interface where you could make requests.

Next on the station machine run the station script, this should notify the lab that the station is online and allow users to send requests via the lab to the station.

The setup is now complete and you can move on to designing and executing workflows using ochra_discovery package.

## Writing and executing a workflow

Assuming you already have a lab and stations set up, and now you want to write a workflow:

First install ochra_discovery by cloning with the command
```bash
git clone https://github.com/OChRA-lab/ochra_discovery.git
```
and installing to your environment with
```bash
pip install ./ochra_discovery
```

Then begin making your workflow script.

The first step in any workflow script is to connect to the lab using the discovery lab class.

```python
from ochra_discovery.spaces.lab import Lab

lab = Lab("lab_ip:8001")
```

To find your target station, use the `get_station` method from the `lab` object, using the station's name provided when setting it up.

```python
# Get IKA station
ika_station = lab.get_station("ika_station")
```

Now, retrieve the needed device. Ensure you have the device package installed in your environment and use the `get_device` method on the `station` object.  

We recommend using type hinting the return object with the correct type to allow your editor to have auto-complete functionalities, making your life easier.

```python
# Get IKA plate
ika_plate: IkaPlate = ika_station.get_device("my_ika")
```

To use one of your device operations, simply call the corresponding method. Each call returns an operation object containing information about what was called and the results.

```python
# Call method, discarding operation
ika_plate.start_stir()

# Call method with arguments
ika_plate.set_speed(100)

# Call method, keeping operation
stop_op = ika_plate.stop_stir()
```

The returned operation object contains the following attributes:

- `caller_id: uuid.UUID`
- `entity_id: uuid.UUID`
- `entity_type: str`
- `method: str`
- `args: Dict[str, Any]`
- `status: OperationStatus = OperationStatus.CREATED`
- `start_timestamp: datetime = Field(default=None)`
- `end_timestamp: datetime = Field(default=None)`
- `result: OperationResult = Field(default=None)`

To see the data produced by an operation, look at the `OperationResult` object:

```python
stop_op = ika_plate.stop_stir()
result = OperationResult(id=UUID(stop_op.result)) 
# Currently, retrieving the result gets its id. In the near future, we will fix this to get the complete result object back 
print(result.result_data)
# Note: Add a method for this in the operation class
```

For more example setups, please check the `ochra_examples` repo.

## UI Integration

There is a way to include UI elements on your servers. This is still a work in progress, but in a nutshell, by using annotated types and decorators, we point variables to Jinja templates and render them as the station starts up.  
This is not a super simple process, and we're still working on it, so we can't fully explain everything at this time.
