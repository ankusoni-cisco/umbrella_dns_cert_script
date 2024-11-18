# Umbrella_dns_cert_script

Issue: 

The SSL digital certificate that is used by Cisco Catalyst SD-WAN Routers to register with Cisco Umbrella DNS expired on September 30, 2024. Cisco SD-WAN Routers with the expired certificate will fail to register with the Cisco Umbrella DNS service. The result of this failure is that all subsequent client DNS requests will not be redirected to Umbrella. 
 
Remediation: 

1. Manual steps: 

     Field notice - https://www.cisco.com/c/en/us/support/docs/field-notices/741/fn74166.html  

 2.  Automation through script: 

      NOTE : USE admin credentials for edge devices 

      Script - script_for_umbrella_rootca.tar.gz.   

        This script is to copy the Umbrella rootCA to all edge routers from vManage through system IP connection. Script does the following steps: 

         a. Fetch all connected reachable routers from vManage.      
         b. Ssh to edge router and copy new Umbrella rootCA to /bootflash/sdwan/trustidrootx3_ca_062035.ca 
         c. Copy root CA file from /bootflash/sdwan/trustidrootx3_ca_062035.ca to bootflash:trustidrootx3_ca_092024.ca 

        This zipped file has four files  

         a. fetch_connected_devices.py - this script fetches all connected router’s system IP and writes it to a csv file (hosts.csv)  
         b. umbrella_rootca_script.py. - this script copies and updates the CA for routers 
         c. trustidrootx3_ca_062035.ca - new root CA file 
         d. host.csv - file with routers system ip, version, username and password in comma separated format (for sample format) 

        Note:  This script works only on single tenant vManage and for vManage version 20.9.x and above.. For cluster deployment, this script has to be run on each vManage node in the cluster. This script will work for edges version 17.03.x or newer with vManage version 20.9.x or newer. 

        Note:  Run the script with maximum of 600 routers in csv file (per vManage node if in cluster deployment).  If there are more than 600 routers per vManage node in cluster, please trigger script run with batch of 600 routers with taking backup of csv as needed. 

Steps to run script: 

      a.   Copy this tar.gz file to vManage path /home/admin/ 

      b.   Ssh to vManage with admin access 

      c.   Enter vshell  

      d.   Unzip the file - “tar -xvf /home/admin/script_for_umbrella_rootca.tar.gz” 

      e.   Run the fetch_connected_devices.py with providing vManage username and password 

            python3 fetch_connected_devices.py -u <username> -p <password> 

      f.   Once fetch_connected_devices.py is run, it will create a  csv file (reachable_hosts.csv) with all connected and reachable routers system IP.  Sample output of hosts.csv is below. 

               vManage:~/script_for_umbrella_rootca# cat reachable_hosts.csv  
                 host,version,username,password 
                 172.16.255.15,17.12.4,, 
                 172.16.255.14,17.9.5a,, 
               vManage:~/script_for_umbrella_rootca# 

          Also above script will create a csv file “unreachable_hosts.csv” if any routers are in unreachable state. 

       g. User has to manually update reachable_hosts.csv file with username and password for each router. Sample output of reachable_hosts.csv is below. 

               vManage:~/script_for_umbrella_rootca# cat reachable_hosts.csv 
                host, version,username,password 
                172.16.255.15,17.12.4,admin,password123 
                172.16.255.14,17.9.5a,admin,pwd12345 
               vManage:~/script_for_umbrella_rootca# 

       h.   Run the root ca update script using following command with right vManage version in the command 

               python3 umbrella_rootca_script.py -c reachable_hosts.csv -v 20.12.3 

       i.   Once the script is run, it will display the results about how many routers are successfully updated and failed. At the end of the run, script will generate failed_routers.csv which will contain system IPs of failed routers if any. Script can be rerun for failed routers using that csv if routers are reachable again. 

       j.   If the username and password is same for all the routers, instead of repeating the username and password for all routers in csv, script can be run with args –u (for username) and –p (for password). CSV file can just have router system IP without username and password. 

               python3 umbrella_rootca_script.py -c reachable_hosts.csv -v 20.12.3 -u <router-username> –p <router-password> 

If you feel that time updating the certificate is taking longer time, we can break the device list in multiples of 100s and can run the script in background. 

Once you get reachable_hosts.csv, run below commands:

This will create 5 CSV files containing 100 devices each: 

```
head -n 1 reachable_hosts.csv > size1.csv
tail -n +2 reachable_hosts.csv | head -n 100 >> size1.csv
head -n 1 reachable_hosts.csv > size2.csv
tail -n +101 reachable_hosts.csv | head -n 100 >> size2.csv
head -n 1 reachable_hosts.csv > size3.csv
tail -n +201 reachable_hosts.csv | head -n 100 >> size3.csv
head -n 1 reachable_hosts.csv > size4.csv
tail -n +301 reachable_hosts.csv | head -n 100 >> size4.csv
head -n 1 reachable_hosts.csv > size5.csv
tail -n +401 reachable_hosts.csv | head -n 100 >> size5.csv

python3 umbrella_rootca_script.py -c size1.csv -v <vManage_version> -u <router_username> -p <router_password> > output_1.log 2>&1 &
python3 umbrella_rootca_script.py -c size2.csv -v <vManage_version> -u <router_username> -p <router_password> > output_2.log 2>&1 &
python3 umbrella_rootca_script.py -c size3.csv -v <vManage_version> -u <router_username> -p <router_password> > output_3.log 2>&1 &
python3 umbrella_rootca_script.py -c size4.csv -v <vManage_version> -u <router_username> -p <router_password> > output_4.log 2>&1 &
python3 umbrella_rootca_script.py -c size5.csv -v <vManage_version> -u <router_username> -p <router_password> > output_5.log 2>&1 &

You can check the process to see if its running:

ps -ef | grep size
```

POST CHECK: 

    If router already hit the DNS registration issue with the expired certificate, then after running the script, please check the umbrella registration on cedge through “show sdwan umbrella device-registration".  

    For cEdges running 17.8.x and older IOS releases, the router has to be reloaded after script run so it can pick the new Umbrella root CA for authentication if DNS security feature configured.  

    For cEdges running 17.9.x or newer IOS releases, router automatically picks up the new Umbrella root CA after script run if Umbrella registration fails and router need not be reloaded. 

 
