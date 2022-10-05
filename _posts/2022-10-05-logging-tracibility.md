# Improving API Visibility with Logs in Go Gin

## Overview
Logging is an integral part of any application. When your API is running in the real world, there are few ways to look at how its functioning and if its running alright. Logs allow us to peek inside the system at the real time and understand if everything is working as expected. The purpose of logging is different for each app and it should be tailored as per the need.

The approach for logging I describe here is for an API that provides data, has a lot of concurrent requests(which API doesnt :P) and interacts with a lot of services in the back. Any API response therefore would be a combination of -
	1. The request data provided by client
	2. Data returned by underlying services and databases
	3. Any computation performed on the data

Logging is very helpful when things break or dont work as expected. The response is a combination of the above 3 factors and anyone might be responsible. Unit and integration tests would cover point 3 with the help of mocking and some aspects of 2. If we add logs at the right places, it would allow us to cover 1 as well and see how the system is running in real time and debug issues if necessary.