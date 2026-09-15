# ignoring IKEv2 request, no policy configured

## Incident
<img width="2075" height="1173" alt="image" src="https://github.com/user-attachments/assets/eff5efd3-bd39-44ff-b8e0-9fac7a24fc09" />

After completing the configuration of the IPsec tunnel on both the PA VM and 100E, I ran a test connection via the PA VM CLI with "test vpn ike-sa gateway to-fortigate" to the 100E and watched the connection attempt live on the 100E, with debug enabled in the CLI. The connection attempt failed, and within the debug information, I saw the lines "Negotiate SA Error" and "ignoring IKEv2 request, no policy configured".

## Troubleshooting Steps

After researching the error lines I saw within the debug output, I came across the following official Fortinet KB article referencing the same exact error. https://community.fortinet.com/fortigate-3/troubleshooting-tip-log-message-ignoring-request-to-establish-ipsec-sa-no-policy-configured-99872

## Root Cause

The root cause of the error messages I saw was that I did not have firewall rules or a static route that referenced the IPsec tunnel at the time.

## Fix

I added firewall rules and a static route to the 100E for the tunnel, as shown in step 5, and the issue was resolved!
