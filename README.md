route -p add 10.138.253.100 mask 255.255.255.255 10.138.204.1 if 14



ping 10.138.253.100
Test-NetConnection 10.138.253.100 -Port 135
