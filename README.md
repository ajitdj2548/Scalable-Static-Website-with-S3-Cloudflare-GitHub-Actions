 
{
    "Statement": [
        {
            "Action": [
                "s3:DeleteObject",
                "s3:GetBucketAcl",
                "s3:GetBucketWebsite",
                "s3:GetObject",
                "s3:GetObjectAcl",
                "s3:GetObjectVersion",
                "s3:GetObjectVersionAcl",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::tutorialstuff.xyz/*",
            "Effect": "Allow",
            "Sid": "S3Access"
        },
        {
            "Action": [
                "codecommit:BatchGetRepositories",
                "codecommit:CreateBranch",
                "codecommit:Get*",
                "codecommit:GitPull",
                "codecommit:GitPush",
                "codecommit:List*",
                "codecommit:Put*",
                "codecommit:Test*",
                "codecommit:Update*"
            ],
            "Resource": [
                "arn:aws:codecommit:us-west-2:XXXXXXXXXXXX:tutorialstuff.xyz"
            ],
            "Effect": "Allow",
            "Sid": "CodeCommitAccess"
        }
    ]
}
```

This policy will allow any IAM user in the group to have the required permissions to manage *only* the static website's S3 bucket and CodeCommit repository.

Now that you understand what the group does it's time to setup the CodeCommmit user configuration. AWS has good documentation on how to configure the IAM user for CodeCommit [here](http://docs.aws.amazon.com/codecommit/latest/userguide/setting-up.html) but I will demonstrate the way I like to do it.

1. Create a key pair using `ssh-keygen` as illustrated in the OS X command line example below. If you haven't done this before get your path with `pwd` so you know exactly where to save your public and private key pair. I like to store mine in a separate location than the default for various reason, one being backup. I like to use a bit size of `4096` but choose what you are comfortable with. For the name of the key pair use whatever make sense to you but in this example I will use the actual IAM user name of `tutorialstuff.xyz-CodeCommitUser-us-west-2` that was created by the CloudFormation stack. Finally, I always use passwords on my SSH keys and I recommend you do the same but make sure you don't store the passphrase with the private key as that will defeat the purpose:

```
> pwd
/Users/virtualjj/Documents/AWS/SSH Keys/tutorialstuff.xyz
> ssh-keygen -b 4096
Generating public/private rsa key pair.
> Enter file in which to save the key (/Users/virtualjj/.ssh/id_rsa): /Users/virtualjj/Documents/AWS/SSH Keys/tutorialstuff.xyz/tutorialstuff.xyz-CodeCommitUser-us-west-2
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
```
<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/codecommitconfig-001-create-4096-keypair.jpg" alt="Create a 4096 bit passphrase protected key pair." height="75%" width="75%">
</p>

View the permissions of the generated public (.pub) and private key pair. Change them to __read-only__ using the `chmod` command:

```
Demo: ls -la
total 16
drwxr-xr-x   4 virtualjj  staff   136 Aug  8 11:18 .
drwxr-xr-x  10 virtualjj  staff   340 Aug  8 11:05 ..
-rw-------   1 virtualjj  staff  3326 Aug  8 11:18 tutorialstuff.xyz-CodeCommitUser-us-west-2
-rw-r--r--   1 virtualjj  staff   745 Aug  8 11:18 tutorialstuff.xyz-CodeCommitUser-us-west-2.pub
Demo: chmod 400 *
Demo: ls -la
total 16
drwxr-xr-x   4 virtualjj  staff   136 Aug  8 11:18 .
drwxr-xr-x  10 virtualjj  staff   340 Aug  8 11:05 ..
-r--------   1 virtualjj  staff  3326 Aug  8 11:18 tutorialstuff.xyz-CodeCommitUser-us-west-2
-r--------   1 virtualjj  staff   745 Aug  8 11:18 tutorialstuff.xyz-CodeCommitUser-us-west-2.pub
```

2. Next click on the IAM user created by this stack and click on the __Security credentials__ tab. Scroll down and select the __Upload SSH public key__ button:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/codecommitconfig-002-upload-codecommit-keypair.jpg" alt="Upload your CodeCommit public key." height="75%" width="75%">
</p>

3. On OS X I like to use a command called `pbcopy` to copy the contents of a file to my clipboard. You can copy the public key to your clipboard like this&mdash;make sure you copy the file with the extension of __.pub__:

```
pbcopy < tutorialstuff.xyz-CodeCommitUser-us-west-2.pub
```
<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/codecommitconfig-003-copy-public-key-to-clipboard.jpg" alt="Use pbcopy to copy public key to clipboard and upload to IAM." height="75%" width="75%">
</p>

You should now have an uploaded CodeCommit public SSH key. Keep note of the __SSH key ID__ as you will need it to configure your local Git repository later:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/codecommitconfig-003-confirm-uploaded-cc-ssh-key.jpg" alt="Confirm that your CodeCommit SSH public key has been successfully uploaded." height="75%" width="75%">
</p>

## SETUP HUGO WEBSITE EXAMPLE

Now that your CodeCommit user has been setup with an SSH key you need to initialize a local repo with `git`. When we performed the steps in [CONFIRM STATIC HOSTING WORKS](#confirm-static-hosting-works) we used a simple index.html file. You'll probably be using a static generator like [Hugo](https://gohugo.io/getting-started/) or [Jekyll](https://jekyllrb.com/) which has a lot of files to track.

1. As an example, I will setup a new Hugo site on my local OS X machine. If you don't have Hugo but want to try it you can follow the [Quick Start](https://gohugo.io/getting-started/quick-start/). Here are the commands to check the Hugo version, create a new site, change directory into it, and confirm your working path:

```
Demo: hugo version
Hugo Static Site Generator v0.25.1 darwin/amd64 BuildDate: 2017-07-13T00:40:37+09:00
Demo: hugo new site s3-tutorialstuff.xyz
Congratulations! Your new Hugo site is created in /Users/virtualjj/Documents/WEBSITES/s3-tutorialstuff.xyz.

Just a few more steps and you're ready to go:

1. Download a theme into the same-named folder.
   Choose a theme from https://themes.gohugo.io/, or
   create your own with the "hugo new theme <THEMENAME>" command.
2. Perhaps you want to add some content. You can add single files
   with "hugo new <SECTIONNAME>/<FILENAME>.<FORMAT>".
3. Start the built-in live server via "hugo server".

Visit https://gohugo.io/ for quickstart guide and full documentation.
Demo: cd s3-tutorialstuff.xyz/
Demo: pwd
/Users/virtualjj/Documents/WEBSITES/s3-tutorialstuff.xyz
```
2. Next list the contents of the `themes` directory and change directory into it:

```
Demo: ls -la themes/
total 0
drwxr-xr-x  2 virtualjj  staff   68 Aug  8 11:49 .
drwxr-xr-x  9 virtualjj  staff  306 Aug  8 11:49 ..
Demo: cd themes/
Demo: pwd
/Users/virtualjj/Documents/WEBSITES/s3-tutorialstuff.xyz/themes
```
3. I will clone the [Aerial theme](https://themes.gohugo.io/aerial/) by [Seth MacLeod](https://www.sethmacleod.com/) and use it as an example:

```
git clone https://github.com/sethmacleod/aerial.git
```

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/setuphugosite-003-download-theme.jpg" alt="Clone the Hugo Aerial theme to the local themes directory." height="75%" width="75%">
</p>

4. Change directory back to the root of the local Hugo site you just created. Copy the Hugo theme's `exampleSite` __config.toml__ to your site's root folder. This file is what configures settings for Hugo and your theme. When you create a new Hugo site locally a config.toml file exists but it won't have the parameters necessary for the theme. The example below shows how I copied and overwrote the default __config.toml__:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/setuphugosite-004-overwrite-config-toml-with-theme-one.jpg" alt="Overwrite the default Hugo new site config.toml with the theme's config.toml configuration file." height="75%" width="75%">
</p>

5. Open the config.toml file that you just copied in your favorite text editor. We need to change:

```
languageCode = "en-us"
title = "Aerial"
baseurl = "http://example.org/"
theme = "aerial"
```

To this&mdash;specifically the `baseurl`. If you don't change the `baseurl` to your domain name the theme links will break and not work properly when you upload the generated site to S3:

```
languageCode = "en-us"
title = "Aerial"
baseurl = "https://tutoriastuff.xyz/"
theme = "aerial"
```

6. Use the following command to run the site locally to confirm that the theme works.

```
hugo server --theme=aerial
```
<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/setuphugosite-006-generate-site-locally.jpg" alt="Generate Hugo site locally to confirm that the theme works." height="75%" width="75%">
</p>

Open up a new tab and go to `localhost:1313`. The local website and theme should display:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/setuphugosite-006-view-local-hugo-site-and-theme.jpg" alt="View the Hugo site locally and make sure the theme is working." height="75%" width="75%">
</p>

7. In your terminal application press `Ctrl+C` to stop the Hugo local server.

## CONFIGURE CODECOMMIT GIT REPO

Now that we have a folder to commit changes to we can configure Git to use the CodeCommit repository that was created with the stack. This section assumes that you aren't too familiar with Git but here is a [cheat sheet](https://www.git-tower.com/blog/git-cheat-sheet/) if you want to explore more.

1. Initialize the Hugo website folder with git:

```
git init
```

Notice that you now have a __.git__ folder listed:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-001-git-init.jpg" alt="Initialize the Hugo website folder with git init." height="75%" width="75%">
</p>

Use the following command to view the Git configuration&mdash;notice there isn't much yet:

```
cat .git/config
```
<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-001-view-git-config.jpg" alt="View the local Git configuration." height="75%" width="75%">
</p>


2. Now we need to add a remote origin&mdash;the CodeCommit repository&mdash;with your IAM user and CodeCommit SSH credentials. Refer back to the `Outputs` section of the launched CloudFormation stack. Look at the one labelled __GitCloneUrlSsh__ and copy the URL to a text editor&mdash;we will make a slight modification:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-002-get-ccommit-url-from-clf-outputs.jpg" alt="Get the CodeCommit URL from the Outputs section of the launched stack." height="75%" width="75%">
</p>

Copy the URL and put your IAM SSH KEY ID that you created in [CONFIGURE CODECOMMIT USER](#configure-codecommit-user) before the CodeCommit URL with an ampersand. We will add the URL to the local Git configuration using the `git remote add origin` command. Here is what mine looks like:

```
git remote add origin ssh://APKAJ2YFIEMJBW6MTT4A@git-codecommit.us-west-2.amazonaws.com/v1/repos/tutorialstuff.xyz
```

3. Execute the full `git remote add origin` command and view the Git config again:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-003-add-remote-origin.jpg" alt="Add the CodeCommit remote origin URL to the local Git configuration." height="75%" width="75%">
</p>

4. Check the status with `git status`; you will see a notification that there are untracked files. Then run `git add .` and commit the changes with `git commit -m "Initial commit."` Finally, try pushing the changes from your local repo to the AWS CodeCommit repo&mdash;it will fail:

```
Demo: git status
On branch master

Initial commit

Untracked files:
  (use "git add <file>..." to include in what will be committed)

	archetypes/
	config.toml
	themes/

nothing added to commit but untracked files present (use "git add" to track)
Demo: git add .
Demo: git commit -m "Initial commit."
[master (root-commit) eafac1b] Initial commit.
 3 files changed, 44 insertions(+)
 create mode 100644 archetypes/default.md
 create mode 100644 config.toml
 create mode 160000 themes/aerial
Demo: git push origin master
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-004-initial-commit-fail.jpg" alt="Observe the failure of the initial commit.="90%" width="90%">
</p>

The reason why this happens is because the CodeCommit repo cannot confirm that you have or are using the correct private CodeCommit SSH key associated with the public key you uploaded your IAM user. Remember that key pair that you generated previously? We need to add the private key (hopefully passphrase protected) to our local SSH identity:

```
ssh-add <your private key file>
ssh-add -L
```

Here is a screenshot of mine&mdash;note that I setup a passphrase on my SSH key so I have to enter it:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-004-add-ssh-key-identity.jpg" alt="Add SSH identity.="90%" width="90%">
</p>

 It's worth mentioning that this method is not what AWS explains to do in their [guide](http://docs.aws.amazon.com/codecommit/latest/userguide/setting-up-ssh-unixes.html). You should be aware that when your SSH identity is added, you won't be prompted for a passphrase each time so that could be a security concern depending on your environment. The threat is that a nefarious individual or bot could obtain shell access to your PC locally or remotely and execute commands since the identity is added to memory. While that is a concern, the likelihood is low and the impact is lower as well since the SSH key only has access to your website S3 bucket and CodeCommit repository. Other than deleting or modifying your public web site in an embarrassing way, there isn't much damage that could be done anyway versus if you were using an IAM user with full administrator access to your AWS account.

 Either way, make sure to remove your SSH identities using `ssh-add -D` when you are done or reboot your computer as a reboot will clear them out as well.

 5. After adding your SSH private key to your local SSH identity, try pushing your local Git repo again. It should push successfully this time:

 <p align="center">
 <img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-005-retry-codecommit-push.jpg" alt="After adding SSH identity trying pushing local Git repo to remote CodeCommit again.="90%" width="90%">
 </p>

 You will also receive an email notification of the activity:

 <p align="center">
 <img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-005-codecommit-sns-email-notification.jpg" alt="After performing an activity on CodeCommit receive an email notification." height="75%" width="75%">
 </p>

6. Go to your AWS console and view the CodeCommit repository. You will be able to see the files that you committed locally now stored on your AWS CodeCommit repository:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/ccconfigure-006-confirm-files-pushed-to-codecommit.jpg" alt="Confirm that local files that were committed have been successfully pushed to AWS CodeCommit." height="75%" width="75%">
</p>

Now the only left to do is copy the files from your statically generated website&mdash;this tutorial's example will be the files in the __public__ folder generated by Hugo.


## COPY STATIC WEBSITE FILES TO S3

Continuing with our Hugo example, go ahead and generate the actual website using the `hugo -v` command. This will create a new folder called __public__:

```
hugo -v
```
<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-generate-hugo-ste.jpg" alt="Generate the Hugo static website to get the files that need to be uploaded to your S3 static website bucket." height="75%" width="75%">
</p>

Remember in [CONFIRM STATIC HOSTING WORKS](#confirm-static-hosting-works) you set your S3 static website bucket policy to public? There are actually two ways to do this. Here is the first way since we've already set the bucket to public.

### DRAG-AND-DROP METHOD

Open the root static website S3 bucket and click the __Upload__ button. Here you can simply drag in the contents of that Hugo public folder. Note that the *index.html* that you previously created if you are following along step-by-step will be overwritten:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-open-for-drag-drop.jpg" alt="Click upload to drag and drop the contents of your Hugo public folder." height="75%" width="75%">
</p>

Drag over the files and press the __Upload__ button:


<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-upload-dragged-dropped-files.jpg" alt="Upload the files and folders that you dragged into the S3 uploader." height="75%" width="75%">
</p>

You can now access your website by DNS name and view your Hugo statically generated site using the Aero theme:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/static-website-accessible-by-dns-name.jpg" alt="Access your website by DNS name to confirm that your Hugo generated site and theme work after uploaded to S3." height="75%" width="75%">
</p>

### CLI METHOD

If you use this method you can upload your statically generated website via the AWS CLI. This might be preferable since you will be at the command line anyway however, you have to generate AWS Access Keys from the console.

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-climethod-create-access-key.jpg" alt="Create and access key so that you can run AWS CLI commands." height="75%" width="75%">
</p>

At the command prompt, run the command `aws configure` to add a profile for this limited IAM user. Use the data from the __Create access key__ for the __Access key ID__ and __Secret access key__. I usually don't store the Secret access key anywhere else (it's stored in plaintext on your local machine) and I don't worry about losing it as I can just disable the lost one and create a new one:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-climethod-access-key-info.jpg" alt="Example of create access key ID and secret dialog box from IAM." height="75%" width="75%">
</p>

Run the local AWS CLI configuration command&mdash;make sure to use a profile especially if you already have a default profile configured. Note that the example below has masked the actual ID and secret:
```
Demo: aws configure --profile tutorialstuff-xyz
AWS Access Key ID [None]: AKIAXXXXXXXXXXXXX
AWS Secret Access Key [None]: AXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
Default region name [None]: us-west-2
Default output format [None]:
```

Now that you have Access keys configured try listing your static website bucket:

```
aws s3 ls tutorialstuff.xyz --profile tutorialstuff-xyz
```
<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-list-bucket-using-aws-cli.jpg" alt="List contents of static website bucket using AWS CLI." height="90%" width="90%">
</p>

This works because the IAM user that you enabled access keys for is part of the group that has a policy action of `s3:GetObject` set to `allow` for this specific S3 bucket.

You can delete the public bucket policy because we will set objects to public when we upload them via the AWS CLI:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-delete-public-bucket-policy.jpg" alt="Delete the public bucket policy and use the AWS CLI instead." height="75%" width="75%">
</p>

Confirm that you cannot access the website anymore (because the bucket policy was deleted):

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-confirm-access-denied-due-to-no-bucket-policy.jpg" alt="When the public access bucket policy is deleted the website will not be accessible anymore." height="75%" width="75%">
</p>

Now this time, when you generate the static website upload the files in the same command:

```
hugo -v && aws s3 sync --acl public-read --sse --delete public/ s3://tutorialstuff.xyz --profile tutorialstuff-xyz
```

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/copystaticfiles-000-generate-site-upload-via-cli.jpg" alt="Generate the Hugo site and upload to S3 setting files to public." height="75%" width="75%">
</p>

The website should be accessible again:

<p align="center">
<img src="https://github.com/virtualjj/aws-s3-backed-cloudflare-static-website/blob/master/images/readme/static-website-accessible-by-dns-name.jpg" alt="Static website accessible by DNS name again." height="75%" width="75%">
</p>

## SUMMARY

I hope this CloudFormation template and tutorial helped you get started with a static website. There are many ways to do this as well as many other tools to automate the deployment of your static websites. However, hopefully you have enough knowledge now to decide what works (and doesn't) for you as well as the confidence to go out and try other methods of deploying static websites on AWS S3 with Cloudflare.

## ACKNOWLEDGMENTS

Thanks to Eric Hammond for inspiring me to work on this project. Much of his [alestic/aws-git-backed-static-website](https://github.com/alestic/aws-git-backed-static-website) template was used as the basis for this project.
