# ETH index=12, GW ETH=10.109.85.20, Wi-Fi index=14, GW Wi-Fi=10.138.204.1

# 1) Internet por Wi-Fi
route delete 0.0.0.0 mask 0.0.0.0 10.109.85.20
route -p add 0.0.0.0 mask 0.0.0.0 10.138.204.1 metric 10 if 14

# 2) Todo 10.109.x.x por ETH
route -p add 10.109.0.0 mask 255.255.0.0 10.109.85.20 metric 5 if 12
