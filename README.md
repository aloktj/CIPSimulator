# CIPSimulator
The CIP Simulator, where CIP stands for Common Industrial Protocol, is a tool that can be run on several platform such as Windows, Linux.
It is a web server application, allowing the user to interact with CIP enabled devices in the network.
The web allows user to create a new CIP configuration, save CIP configuration, reset configuration, set previously saved configuration.
The web also allows user the provision to change the payload of each CIP I/O data.
On the backend, the CIPSimulator uses https://github.com/EIPStackGroup/OpENer.git as a submomdule dependency, which is the CIP stack for CIP communication.
