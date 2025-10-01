# Setup Student VM's

# Clone the lab repo in VS Code

### Step 1: Open the folder

1. Open Visual Studio Code
2. In Visual Studio Code, click **Clone Repository** and paste `https://github.com/jruels/e2e-automation.git`
3. Hit **Enter**, and in the pop-up window, browse to `C:\Users\tekstudent\Downloads\repos`
4. Click **Select as repository destination**
5. When prompted to open the cloned repo, choose **Open**.
6. After opening the folder, click the third icon in the left toolbar for source control. Next to **changes**, click the ellipses (three dots) and choose **pull**.

# Configure AWS credentials

### **Step 1: Log into the AWS Console**

1. In a browser, log into the [AWS Console](https://console.aws.amazon.com/) using the credentials in the spreadsheet below.
    * [Cloud credentials](https://docs.google.com/spreadsheets/d/1qWbTS_0iieOvcZzA37Ent3WYAX2rHiSpJMGS6wbCtMk/edit?usp=sharing)
2. Search for IAM in the search bar.
3. Click **IAM**
4.Click **Users**.
5. Click **autodev-admin**.
6. Click **Security Credentials**.
7. Scroll down to **Access Keys**
   1. Click **Actions** -> and **Deactivate** and **Delete** any existing keys.
8. Click **Create access key**
9. Select **Command Line Interface (CLI)**. 
10. Check the confirmation box at the bottom of the page and click **Next**.
11. Skip the description and click **Create access key**.
12. **IMPORTANT:** Copy the **Access key** and **Secret access key** and save them somewhere. You can optionally download the `csv` file for easy reference. 

---

### **Step 2: Use AWS Configure**

1. In the Visual Studio Code Terminal run: 

   ```
   aws configure
   ```

2. Supply the required information.
   * Credentials 
   * Region = `us-west-1`
3. Select the default option for the remaining options.

---

### **Step 3: Test AWS CLI**

1. Confirm that `aws` can use the credentials.

   ```
   aws sts get-caller-identity
   ```

2. You should see your account information returned.



## Congratulations

Congratulations, you've successfully configured your machine.
