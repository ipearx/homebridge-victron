OK after a year or two of procrastinating on this, I finally finished how to control a Victron inverter from Homekit using Homebridge.
You will need:

- A raspberry pi with Homebridge installed and running on it.
- A Victron Colour Controller or other VenusOS device.
- Modbus enabled on the Venus device.
- The cmd4 Homebridge plugin.
- Node installed on your pi, and jsmodbus installed (with npm install jsmodbus)

My configuration for the Homebridge cmd4 plugin is:

```JSON
{
    "platform": "Cmd4",
    "name": "Cmd4",
    "outputConstants": false,
    "accessories": [
        {
            "type": "LightSensor",
            "displayName": "Battery",
            "statusActive": "TRUE",
            "currentAmbientLightLevel": "0",
            "name": "Battery",
            "polling": true,
            "state_cmd": "node /home/pi/victronmodbus/battery.js"
        },
        {
            "type": "Switch",
            "displayName": "Inverter",
            "on": "FALSE",
            "name": "Inverter",
            "polling": true,
            "state_cmd": "node /home/pi/victronmodbus/inverter.js"
        }
    ]
}
```

There are two scripts. Copy to your raspberry pi, I put in a folder called "victronmodbus" in the pi home folder. You'll need to configure this to read the data from the hardware you've got. You can get a spreadsheet from victron that will describe how to get data off different devices: https://www.victronenergy.com/support-and-downloads/technical-information
My battery level script is as follows.
To use you'll need to change the IP address of your VenusOS device. 245 is the device ID, you can find this in your VenusOS device under the Modbus menu. 266 is the register to read, found in the spreadsheet.

```Javascript
const Net = require('net')
const Modbus = require('jsmodbus')

//create a tcp modbus client
const bm_socket = new Net.Socket()
const batterymonitor = new Modbus.client.TCP(bm_socket, 245)
const options = {
	'host' : '192.168.88.3',
	'port' : 502
}
bm_socket.on('connect', function () {
	batterymonitor.readInputRegisters(266, 1).then(function (resp) {
		//console.log(resp);
		console.log(Math.floor(resp.response.body._values[0]/10));
		bm_socket.end();
		process.exit( 0 );
	}, console.error);
});
bm_socket.connect(options);
```


And my inverter script is:


```Javascript
const Net = require('net')
const Modbus = require('jsmodbus')

// state_cmd Set < DisplayName > < characteristic > < value >
const myArgs = process.argv.slice(2);

const bm_socket = new Net.Socket()
const inverter = new Modbus.client.TCP(bm_socket, 246)
const options = {
	'host' : '192.168.88.3',
	'port' : 502
}

switch (myArgs[0]) {
	case 'Get':
		// check inverter is on

		bm_socket.on('connect', function () {
			inverter.readInputRegisters(33, 1).then(function (resp) {
				bm_socket.end();
				switch (resp.response.body._values[0]) {
					case 3: 
					case 2: 
					case 1:
						console.log(1); 
						break;
					case 4:
						console.log(0); 
						break;
				}
				process.exit( 0 );
			}, console.error);
		});
		bm_socket.connect(options);
		break;

	case 'Set':
		if (myArgs[3]=="0") {
			// turn inverter off
						console.log('turn off'); 
			bm_socket.on('connect', function () {
				inverter.writeSingleRegister(33, 4);
				process.exit( 0 );
			});
		} else {
			// turn inverter on
						console.log('turn on'); 
			bm_socket.on('connect', function () {
				inverter.writeSingleRegister(33, 3);
				process.exit( 0 );
			});
		}
		bm_socket.connect(options);
		break;
}
```


Hope that helps someone! Maybe me in the future when I google this again...
