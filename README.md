# How to get Started Farming for Chia Coin in AWS

This is a detailed how-to guide that will clearly outline every step invovled with creating your farming setup. <br />

AWS offerings used: `EC2 instance`, `S3 bucket`, `IAM`

# Steps: 

1) Setup AWS account 
2) Launch `EC2 instance`
   - Choose `i3.xlarge` w/ `Amazon Linux Image`
   - Specify `root volume` as `30 GB` 
   - Make sure when you get to `Step 6:Configure Security Group`, you select `My IP` under the `Source` tab for `SSH` protocol. 
   - Request to generate a `.pem` key pair and store it in `Downloads` of local computer 
   - Click `Launch Instance`

3) Navigate to your instance in AWS console and once it's ready, select your instance and hit `Connect`. Copy the `SSH` example command under the `SSH Client` tab. Go to `Downloads` directory via terminal and paste that command in. 

Note: If you get `Unprotected Key File Warning`, then open terminal at `Downloads` directory and run: <br />
`chmod 400 nameOfFile.pem` (This should be where your `.pem` key-pair file is located)

4) Now you are logged into EC2 instance. 
5) Run `sudo fdisk /dev/nvme0n1`
6) This will launch the fdisk utility program. Type `p` to see existing partition table. Now, type `n` to create a new partition. Choose `primary`. Then, click enter thru the next steps till it says `Last sector`. For `Last sector`, type `+100G` to give this partition `100 GB` space. 
7) We will create one more partition which will store the rest of this server's storage in a partition. While staying in the `fdisk` utility program, type `n` and hit `enter` on keyboard (default options) for the all the options. Make you select `primary` for this partition as well.

8) Now hit `p`. This should show a list of partitions. 

```
/dev/nvme0n1p1       Size: 100 GB
/dev/nvme01n1p1      Size: 768 GB
```

10) Now format the partition to the `xfs` filesystem. <br />
`sudo mkfs -t xfs /dev/nvme0n1p1`

11) Now create a mount point for the partition. <br />
`sudo mkdir -p /tmp1` 

12) Now mount the partition. <br />
`sudo mount /dev/nvme0n1p1 /tmp1`

13) Type `df -hT` and check that you see the `/dev/nvme0n1p1` partition mounted on `/tmp1`

Output should have: <br />

```
Filesystem      Type      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1  xfs       100G   94G  6.6G  94% /tmp1
```

14) Now modify the permissions of the mount point to `ec2-user`: <br />
`sudo chown -R ec2-user.ec2-user /tmp1` 

15) Now go to your AWS Console and create an S3 Bucket. (Note down the name you give to the bucket)

16) Install `goofys` on EC2 host. <br />
`wget https://github.com/kahing/goofys/releases/latest/download/goofys` <br />
`chmod u+x goofys` <br />
`mkdir /home/ec2-user/chia` <br />

17) Now go back to AWS Console. And go to `IAM`. Click on `Policies` and create new policy. Paste in this `JSON`: 

Note: Change `nameOfS3Bucket` to the name of your S3 Bucket. <br />

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "s3:ListBucket",
            "Resource": [
                "arn:aws:s3:::nameOfS3Bucket"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:DeleteObject"
            ],
            "Resource": [
                "arn:aws:s3:::nameOfS3Bucket/*"
            ]
        }
    ]
}
```

18) Click thru the default options and finally create the policy. Now go to `Users`. And hit `Add User`. 

19) Click on `attach existing policies directly`. Filter for `Customer managed policies` and select the one you just created. Now click thru the options to create a new user. 

20) Upon successful completion, you will see the created user's `Access key ID` and `Secret Access Key`. Note these down in a notepad. 

21) Go back to your terminal that's still logged into your EC2 instance. 

22) Switch to root and create AWS credentials file. <br />
`sudo su -root` <br />
`mkdir ~/.aws` <br />
`vi ~/.aws/credentials`    (Using Vim terminal text editor) <br />

Click `I` key on your keyboard to start writing in this text file. Paste ALL this in: 

```
[default]
aws_access_key_id = AWS_ACCESS_KEY_ID
aws_secret_access_key = AWS_SECRET_ACCESS_KEY
```

Note: Replace the capitalized stuff with your actual credentials from step 20. 

When finished editing the file, exit and save by doing: <br />
Hit `Escape` key on keyboard once. 
Type `:wq` and hit `Enter` on keyboard. 

At this point switch from root user to regular by typing: <br />
`sudo su - ec2-user` 

23) Now we create a mount point. <br />
`sudo mkdir -p /home/ec2-user/chia`

24) Run: <br />
`sudo ./goofys --uid 1000 --gid 1000 -o allow_other nameOfS3Bucket /home/ec2-user/chia`

Make sure to replace `nameOfS3Bucket` with the name of your S3 Bucket. 

25) Type `df -h` and make sure your output has this: <br />

```
Filesystem      Type      Size  Used Avail Use% Mounted on
/dev/nvme0n1p1   100G   91G  9.2G  91% /tmp1
nameOfS3Bucket  1.0P     0  1.0P   0% /home/ec2-user/chia
```

26) Install `Chia` code <br />
`sudo yum update -y` <br />
`sudo yum install python3 git -y` <br />
`git clone https://github.com/Chia-Network/chia-blockchain.git -b latest —recurse-submodules` <br />

27) Set up `Chia` <br />
`cd chia-blockchain` <br />
`chmod +x ./install.sh` <br />
`sh install.sh` <br />
`. ./activate` <br />

28)  Install and execute the initialization command for the first time: <br />
`chia init` 

29) If this is your first time, run this to generate a new wallet address: <br />
`chia keys generate` 

Note: If you already have a wallet, you can type `chia keys add` and paste in your generated mnemonic. <br />

Make sure you note the output of this command safely. <br />

30) Start the `Chia` process. <br />
`chia start farmer` 

31) Start farming <br />
`nohup chia plots create -k 32 -b 6000 -r 2 -n 2 -t /tmp1 -2 /tmp1 -d /home/ec2-user/chia>>plots1.log 2>&1 &`

You are now on your way to farming! <br />

32) Run this to see stats about your farm: <br />
`chia farm summary` 

Note: The first time you run it, you may see some `Exception`. The second time, this will go away. 
Basically, you are not farming yet. The system will say `Farming status: Syncing`, because it is syncing
with the `Chia` blockchain. It needs to become a serviceable node on the blockchain before it can start plotting. 

33) That's it! You are done setting up! You can close your terminal, if you'd like. <br />


# How to Quickly Check in on your Farm: <br />

If you would like to see the status of your farming setup, you can go to `Downloads` via 
the terminal. And paste in the `SSH` command you got in Step 3. Once logged into the server, type: <br />
`cd chia-blockchain` <br />
`. ./activate` <br />
`chia farm summary` <br />

To see the status of your `Chia` wallet, run: <br />
`chia wallet show` 

To exit the AWS EC2 instance from your terminal, simply type: <br />
`exit` 

Now you're back home. Enjoy!

# References: <br />
<br />
Setup: <br />
https://newdaycrypto.com/how-to-mine-chia-cryptocurrency-on-amazon-aws-tutorial/ <br />
https://zhuanlan.zhihu.com/p/370431286 <br />
<br />
S3 Bucket and IAM User & Policy: <br />
https://m.blog.daum.net/techtip/12415315?tp_nil_a=2 <br />
<br />
Linux Knowledge: <br />
https://phoenixnap.com/kb/linux-create-partition <br />
https://linuxize.com/post/fdisk-command-in-linux/ <br />
<br />
Debugging: <br />
https://askubuntu.com/questions/1096849/cant-make-new-dir-with-mkdir <br />
https://99robots.com/how-to-fix-permission-error-ssh-amazon-ec2-instance/ <br />








