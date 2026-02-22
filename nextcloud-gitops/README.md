Perfect! Now I see the issue - you're accessing via the EC2 public hostname, but your trusted_domains only has the private IP 10.0.1.218. Let's fix this:

🚀 Add EC2 Public Hostname to Trusted Domains
bash
# Add the EC2 public hostname to trusted domains
kubectl exec -n nextcloud -it deploy/next-cloud-nextcloud -- \
  php occ config:system:set trusted_domains 2 --value="ec2-13-218-232-56.compute-1.amazonaws.com"

# Also add the public IP if needed
kubectl exec -n nextcloud -it deploy/next-cloud-nextcloud -- \
  php occ config:system:set trusted_domains 3 --value="13.218.232.56"

# Verify all trusted domains
kubectl exec -n nextcloud -it deploy/next-cloud-nextcloud -- \
  php occ config:system:get trusted_domains
Expected output:

text
localhost
10.0.1.218
ec2-13-218-232-56.compute-1.amazonaws.com
13.218.232.56
🌐 Now Access via:
text
http://ec2-13-218-232-56.compute-1.amazonaws.com:32538
This should work immediately once the config is updated.

🔧 If Still Having Issues
Check if the pod needs restarting
bash
# Restart the deployment to pick up config changes
kubectl rollout restart deploy -n nextcloud nextcloud

# Wait for new pod
kubectl get pods -n nextcloud -w
Verify the NodePort is accessible from outside
bash
# Check security group rules (from your AWS console or CLI)
aws ec2 describe-security-groups --filters Name=group-name,Values=*node* --query 'SecurityGroups[*].IpPermissions'

# Ensure port 32538 is open to your IP (0.0.0.0/0 for testing)
Test connectivity
bash
# From your local machine
curl -I http://ec2-13-218-232-56.compute-1.amazonaws.com:32538

# Should return HTTP 200 or 302 (redirect to login)
📊 Your Current Setup
Component	Value
Nextcloud Pod	Running on node 10.0.1.218
Service Type	NodePort
Port	32538
Node IP	10.0.1.218
Public Hostname	ec2-13-218-232-56.compute-1.amazonaws.com
Public IP	13.218.232.56
🎯 Quick One-Liner to Fix
Run this single command to add both hostname and IP:

bash
kubectl exec -n nextcloud -it deploy/next-cloud-nextcloud -- bash -c "\
  php occ config:system:set trusted_domains 2 --value='ec2-13-218-232-56.compute-1.amazonaws.com' && \
  php occ config:system:set trusted_domains 3 --value='13.218.232.56' && \
  php occ config:system:get trusted_domains"
After this, refresh your browser at http://ec2-13-218-232-56.compute-1.amazonaws.com:32538 - you should see the Nextcloud login page!

Login with:

Username: admin

Password: (get it with the secret command from earlier)
