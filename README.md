

# AWS Systems Manager: Fleet Rightsizing Automation

## 📖 Scenario
I simulated enterprise environment spanning 3 regions with 10,000 instances, a cost audit identified 2,500 instances as over-provisioned. The objective was to resize these instances (t3.small → t3.micro) efficiently without manual intervention.

## ⚙️ Architecture & Tools
* **AWS Systems Manager (SSM) Automation:** Used to orchestrate the workflow.
* **Runbook:** `AWS-ResizeInstance`
* **Amazon EC2:** The target compute resources.
* **Tagging Strategy:** Used as the primary filter for automation targeting.

## 🚀 Workflow Execution
The automation workflow followed these steps:
1.  **Identification:** Selected targets based on tag `Name: Test-Instance`.
2.  **Pre-Flight Check:** Verified current instance type is not already `t3.micro`.
3.  **Stop:** Issued `StopInstances` API call.
4.  **Resize:** Issued `ModifyInstanceAttribute` to update instance type.
5.  **Start:** Issued `StartInstances` API call.
6.  **Verification:** Validated successful state transition.

## 💡 Key Learnings
* **Rate Control:** Learned to throttle automation execution (e.g., handling 10% of fleet at a time) to maintain service availability.
* **Idempotency:** The importance of the `aws:assertAwsResourceProperty` step to prevent the automation from trying to resize instances that are already correct.




<img width="1608" height="842" alt="Screenshot 2025-12-27 at 8 48 59 PM" src="https://github.com/user-attachments/assets/31a99bdd-f7fb-4e8c-8746-7a3ffe9a9685" />

<img width="1623" height="408" alt="Screenshot 2025-12-27 at 8 51 33 PM" src="https://github.com/user-attachments/assets/01a84ac5-9c4d-48de-8af6-3f500ed838ce" />

<img width="1621" height="817" alt="Screenshot 2025-12-27 at 8 53 10 PM" src="https://github.com/user-attachments/assets/9c30d843-c687-4438-a4a5-4b9e1ac16dcf" />

<img width="1674" height="357" alt="Screenshot 2025-12-27 at 8 54 11 PM" src="https://github.com/user-attachments/assets/d3dea392-1114-45af-821d-02976f69a2fe" />


* ## 🔧 Troubleshooting & Common Issues

During the execution of the `AWS-ResizeInstance` runbook, the following issues may occur. Here is how to diagnose and resolve them:

### 1. Automation Execution Finds "0 Targets"
* **Symptom:** The automation status immediately switches to *Success* or *Failed*, but no instances were modified. The "Total" targets count is 0.
* **Cause:** This usually stems from a mismatch in **Tag filters**. AWS Tags are case-sensitive.
* **Resolution:** Verify the target instances have the exact tag Key (`Name`) and Value (`Test-Instance`). Ensure no trailing spaces exist in the tag values.

### 2. Execution Status Stuck on "Pending" or "In Progress"
* **Symptom:** The automation step `aws:changeInstanceState` (Stop) hangs for an extended period.
* **Cause:** The EC2 instance may be running a process that is preventing a clean shutdown, or the OS is unresponsive.
* **Resolution:** Check the specific instance's system log in the EC2 console. If the instance cannot stop gracefully within the timeout window, the automation will fail. A manual "Force Stop" may be required (though this risks data corruption).

### 3. Error: "Client.UnsupportedOperation"
* **Symptom:** The `aws:executeAwsApi` (ModifyInstanceAttribute) step fails.
* **Cause:** You are attempting to resize to an instance type that is not compatible with the current AMI (e.g., trying to switch from an x86 instance to an ARM-based `t4g` instance without changing the AMI).
* **Resolution:** Ensure the target `InstanceType` is compatible with the instance's architecture (x86_64 vs. ARM64).

### 4. IAM Permission Errors
* **Symptom:** Error message `Client.UnauthorizedOperation` appears in the Execution Detail.
* **Cause:** The IAM User or Service Role executing the automation lacks the required permissions (`ec2:StopInstances`, `ec2:ModifyInstanceAttribute`, `ec2:StartInstances`).
* **Resolution:** Review the IAM policy attached to the Automation Service Role and ensure it follows the principle of least privilege while allowing these specific actions.
