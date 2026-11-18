# Automated-Server-Backup-System-on-Microsoft-Azure
In this repo while doing the project I will be documenting my thought process why i choose a decission and also explain some parts of the working. 

## What I am building 
I am assuming that i am the employee of a company which hosts its website and has given me a role for managing cloud infra of the particular company. The manager comes one morning and say this exact words to me 

Manager:
> Hey Rajvardhan, the website our cokmpany recently build is running up all good but our senour software architect said that it currently does not have a backup system so if anything goes wrong we could be at big risk as you know our most of the clients and projects all are handled with the website so i want you to build a system which can handle the requirement and backup all the files of the website.

Me: 
> Ok boss. I will build the required solution till 6pm today dont warry. And to be fair this was also in my mind back then when i created and provisioned the VM its good they pointed out this early.

Manager:
> Nice, please report me this after you finish and this is a urgent work so leave other work and start doing it now ok.

-----

Self thought:
> Ok so they only have a VM and they even dont have any RBSC and security is also problem here  so i think i would do both the task this time at same time.

I started by investigating in which directory did they hosted their eb project and i found out it was in `/var/www/ImaginaryCompanyLtd` 

so here what i plan to  do first I will create a storage account and in it create a container . And because it has backups we will use lifecycle to delete the backups after 90 days as this is just in case if something got corrupts on server we could restore the files. I will enable the versioning and soft delete both.

I will give the vm access to the storage with Contributor blob data role so it can access without managing some keys seceretly and then rolling them .

after i will do this i will on the create a archieve of the folder and then upload it with the azcopy with the today date and i am thinking to upload it like setup a cron job to run after every 24 hour.

Once we did this we would need to test the features we developed be breaking sokmethings Now for testing i think i would do that on a test vm instead of a real production but sience this is a demo project so i willl be dong the test on the real machines 
1. I will delete a backup to test if i can safely retrive it within the soft delete time period
2. I will delete some files on the web server and try to download the backup form storage to restore all the changes.

### The steps

1. Create the storage account
   `bash az storage account create -n icltd74372173hf7454fg --access-tier cold --sku Standard_LRS`
   I know in real world i should have choosen GZRS as it is more reliable but for my small project need LRS was enough. I should have also choosen the hot tier but then i thought backup corrupts dont occur every day they occur hardly in a mounth so cold tier which i think offer 90 days of storage needs is good.

2. Create the container inside the storage
   `bash az storage container create -n server-backups --auth-mode login --public-access off`
   create the continer with public access off

3. The above command went wrong so i did some -h to figure out what went wrong and found i also need to give the name of the storage account
    `bash az storage container create -n server-backups --auth-mode login --public-access off --account-name icltd74372173hf7454fg`

4. Now was time for RBAC and trust me its my first time I did try it using command line but something was wrong every command was now working as expected so then  i did that from the portal this is the steps i follow
    Portal -> Storage Account -> IAM -> Role assignment -> Selected the VM -> assigned the role of `Storage Blob Data Contributor`

OK so things are getting shape 

Now in real world case I would have already created the VM but here sience this is a LAB i need to create it now so here is the code i written in bash to create it all together

```bash
az configure --defaults location=indiasouthcentral # I dont like writting this all time there for eused this it is a good command time saver

az vm create \
  --name vm-webapp-prod-01 \
  --image Ubuntu2204 \
  --size Standard_B2ats_v2 \
  --os-disk-size-gb 64 \
  --storage-sku Premium_LRS \
  --admin-username rajvardhan \
  --generate-ssh-keys          # Create the VM Notice the naminc scheme here and rest the parameters here are self explanatory
```

Thats it. no its now

Now comes the parts Nginx
Sience this is my first blog i need to tell you that i have used nginx during my application development learning period in my second year of college therefore it was not a new thing but it is still confusing to configure it 

For the part of the fake websites which we need to host i created a demo website of html and css quick using AI tools and changed the configuration of the sites-available. 
Now here i need to be fair I used AI help as even after trying for hours what waent wrong i ask ai to generate a script so it setup the config files correctly as hosting was outside the domain of this lab so i guess i was allowed to use only for that part

Now website was live at have a look how it look 

<img width="1918" height="1033" alt="image" src="https://github.com/user-attachments/assets/f4a55543-3253-4135-909c-7b1f6d3dd2e4" />

It look very basic but ok to work

Also i needed to setup some networking as the Netowrk security group which was assigned to the NIC  does not allow http so i changed it to allow 

```bash
az network nsg rule create \
  --nsg-name $vm01nsg \
  -n "Allow-HTTP" \
  --priority 100 \
  --access Allow \
  --source-address-prefixes "*" \
  --destination-port-ranges 80 \
  --protocol Tcp \
  --description "Allow http traffic to port 80 from any ip address"
```
then i enabled the blob versioning and soft delete feature

```bash

az storage account blob-service-properties update \
  --account-name icltd74372173hf7454fg \
  --enable-versioning true


az storage account blob-service-properties update \
  --account-name icltd74372173hf7454fg \
  --enable-versioning true \
  --enable-delete-retention true \
  --delete-retention-days 30


az storage account blob-service-properties update \
  --account-name icltd74372173hf7454fg \
  --enable-change-feed true
```


Script which i wrote which upoads the website files to backup storage

```bash
#!/bin/bash

# Website directory
directory="/var/www/html"


todayDate=$(date +%F)


tar -czvf "backup-$todayDate.tar.gz" "$directory"


azcopy login --identity


azcopy copy \
  "./backup-$todayDate.tar.gz" \
  "https://icltd74372173hf7454fg.blob.core.windows.net/server-backups/backup-$todayDate.tar.gz"
```











