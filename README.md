# IoT Temperature Monitoring: From sensor to reporting dashboard

## The problem
In restaurants, there are often thousands of dollars worth of food stored in temperature controlled environments like walk-in coolers and freezers. From time to time, the components in these environments can fail when nobody is in the store and that food can be destroyed leading to product loss and significant cost in replacement, even if insurance is used. 

## A solution
Temperature monitoring system that automatically reports the temperature to a dashboard and contacts the right people when things go wrong. 

**Why build this when products exist on the market?** Good question. The answer speaks to the values of the client I built this system for. The client manages about 200 temperature controlled environments, most of them for dough proofing, the rest for perishable ingredients (walk-ins). 
This client needed
* **Cost control**--Building each of the components (detailed later) from parts allows the client to:
  - Control costs at all levels of implementation, 
  - Choose when and how to pivot if prices rise
  - Repair, rather than replace, broken broken components
* **Reliability in reporting**--Most products on the market are battery powered.
  - While they do tend to last a long time, when you are managing 200 different sensors scattered across multiple geographic locations, ensuring batteries are supplied can quickly get out of hand.
  - Building the system in-house allows the client to power the system exactly how they need to (we built it to be powered by 5v power adapters)   
* **A product to share**--In addition to their own retail environments, the client has relationships with several other partners and the client wanted to offer this product to those partners, and wanted to be sure they trusted the product. 

Enter the temperature alert system I built, starting with the hardware of temperature gathering probes, and moving through the design and development process all the way to a dashboard where temperature, sensor health, users, and notification subscriptions are managed. 

## Design 
* [Micropython] A temperature sensor paired with a microcontroller (spoke) reads the temperature and immediately sends it to -> 
* [Python & flask, MariadDB] A single board computer (hub) gathers temperature data and stores it locally in a database, sending it to  ->
* [LAMP] A centralized server gathering temperature data, interpreting the data for warnings, hosting a dashboard to view and analyze trends, and tweak notification settings 
### Central Server & Dashboard
#### Hardware
Debian VM on GCP
#### Software
* LAMP Stack (Debian, Apache2, MariaDB, PHP)
* OAuth 2.0 via Google Workspace + Cloud deployment 
* Temperature monitoring dashboard 
* Location management, define sites (store 432) and locations within sites (Cooler 2)
* Hub management, temperature REQUESTS are only accepted from hubs that have authorization and valid site associated with them
* Spoke management, assigning which location within their reporting site they are taking temperature data
* Threshold management, determining which temperatures are out of bounds for which sensor types  
* User management, who can access the dashboard, which sites they can see, and what type of temperature dashboard should they get (individual store, “DM”/Subset of company, or “whole company”)

### Hubs
#### Hardware
Single board computer, anything that will run command line linux. [Le Potato](https://libre.computer/products/aml-s905x-cc/) was used for this project. 
#### Software
* Convert the wireless radio to broadcast a private IoT network which does not access the internet, only used for spokes to transmit temperature data
* Flask API endpoint to receive temperature data and place it in MariaDB table. Basic API authentication with a bearer token transmitted by spokes to slow down malicious actors
* Database watcher looking for unsent data, run via cron 

### Spokes
#### Hardware
* SHT40 Temperature & Humidity Sensor 
*  RP2040 Microcontroller 
* Waterproof junction box
* Wire glands  
#### Software 
Micropython
Boot script which:
* Checks for network connection to hub
* Reads temperature data with this [firmware by Pjurek3](https://github.com/Pjurek3/sht40_micropython)
* Immediately transmit the data the hub, nothing stored locally (local flat file storage was tested but was not reliable due to the lack of on-board clock)
* Sleep, repeat
* After (x) runs, perform full reboot 
* Any errors causes a sleep for (x) seconds, and then a reboot
