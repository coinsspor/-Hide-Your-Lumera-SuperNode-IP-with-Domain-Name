# 🔒 Hide Your Lumera SuperNode IP with Domain Name

## Overview
Replace your SuperNode's public IP address with a domain name using Nginx reverse proxy for better privacy and security.

## Prerequisites
- Running Lumera SuperNode
- Ubuntu 22.04 LTS
- A domain name
- Root access to your server

## Step 1: Configure Your Domain DNS

In your domain provider's DNS panel:
1. Create an **A Record**
2. Set **Name** to your subdomain (e.g., `supernode`)
3. Set **Value** to your server's IP address
4. Set **TTL** to 300 seconds
5. If using Cloudflare, disable proxy (grey cloud icon)

## Step 2: Install Nginx

```bash
sudo apt update
sudo apt install nginx -y
```

## Step 3: Create Nginx Configuration

Create a new configuration file for your SuperNode:

```bash
sudo nano /etc/nginx/sites-available/supernode
```

Add the following configuration (replace `supernode.yourdomain.com` with your actual domain):

```nginx
server {
    listen 80;
    server_name supernode.yourdomain.com;

    # gRPC proxy for SuperNode
    location / {
        grpc_pass grpc://127.0.0.1:4444;
        error_page 502 = /error502grpc;
    }

    location = /error502grpc {
        internal;
        default_type application/grpc;
        add_header grpc-status 14;
        add_header grpc-message "unavailable";
        return 204;
    }
}
```

## Step 4: Enable the Configuration

```bash
# Create symbolic link to enable the site
sudo ln -s /etc/nginx/sites-available/supernode /etc/nginx/sites-enabled/

# Test the configuration
sudo nginx -t

# Restart Nginx
sudo systemctl restart nginx
```

## Step 5: Install SSL Certificate

Install Certbot for Let's Encrypt SSL:

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get SSL certificate (replace with your domain and email)
sudo certbot --nginx -d supernode.yourdomain.com
```

When prompted:
1. Enter your email address
2. Agree to terms (type 'A')
3. Choose whether to share email with EFF (N for no)
4. Select option 2 to redirect all traffic to HTTPS

## Step 6: Update Nginx for HTTPS

After SSL installation, Nginx will automatically update your configuration. To add HTTP2 support for better gRPC performance:

```bash
sudo nano /etc/nginx/sites-available/supernode
```

Find the line `listen 443 ssl;` and change it to:
```nginx
listen 443 ssl http2;
```

Save and restart Nginx:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

## Step 7: Update Your SuperNode Registration

Now update your SuperNode endpoint in the blockchain to use your domain:

```bash
lumerad tx supernode update-supernode \
  <your-validator-operator-address> \
  supernode.yourdomain.com:443 \
  1.0.0 \
  <your-supernode-account-address> \
  --from <your-wallet-name> \
  --chain-id lumera-testnet-2 \
  --gas auto \
  --gas-adjustment 1.3 \
  --fees 10000ulume -y
```

**Note:** The update command requires 4 parameters:
1. Validator operator address
2. New endpoint (your domain:443)
3. Version (use 1.0.0)
4. SuperNode account address

## Step 8: Verify the Update

Check if your domain is now registered:

```bash
lumerad query supernode get-super-node <your-validator-operator-address>
```

Look for your domain in the `prev_ip_addresses` field.

## Testing Your Setup

Test if your domain is working:

```bash
# Test HTTPS connection
curl -I https://supernode.yourdomain.com

# You should see:
# HTTP/2 204
# grpc-status: 14
# grpc-message: unavailable
```

## Troubleshooting

### DNS not resolving
Wait 5-10 minutes for DNS propagation. Test with:
```bash
ping supernode.yourdomain.com
```

### Port conflicts
If you get port binding errors, check what's using the port:
```bash
sudo netstat -tulpn | grep :443
```

### SSL certificate issues
Renew certificate manually if needed:
```bash
sudo certbot renew
```

## Result

**Before:** Your SuperNode shows as `XX.XX.XX.XX:4444` in the explorer

**After:** Your SuperNode shows as `supernode.yourdomain.com:443` in the explorer

## Important Notes

- ✅ Your actual IP is now hidden behind the domain
- ✅ All traffic is encrypted with HTTPS
- ✅ SSL certificates auto-renew every 90 days
- ✅ You can add Cloudflare later for DDoS protection

## Need Help?

- Check Nginx logs: `sudo journalctl -u nginx -f`
- Check SSL status: `sudo certbot certificates`
- Verify ports are open: `sudo ufw status`

---

**Created for Lumera SuperNode Operators**
