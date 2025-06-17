start by installing paperless.ngx using the script in:
https://community-scripts.github.io/ProxmoxVE/scripts?id=paperless-ngx

Installing Apache Tika and Gotenberg for Paperless-ngx in a Debian LXC ContainerThis document outlines the detailed steps to set up Apache Tika and Gotenberg within a Debian LXC container for a bare-metal Paperless-ngx installation. Tika handles text extraction from Office documents and emails, while Gotenberg converts these documents to PDF for proper processing by Paperless-ngx.PrerequisitesA running Debian LXC container with Paperless-ngx already installed (bare-metal, not Docker for Paperless-ngx itself).sudo access within the LXC container.Java Runtime Environment (JRE) 17 or higher installed in the LXC container.If not already installed, run: sudo apt update && sudo apt install -y openjdk-17-jre1. Install and Configure Apache Tika ServerTika is essential for extracting text content from file formats like .docx, .xlsx, and .eml.1.1 Download Tika Server JARWe will use a stable 2.x version of Tika. As of this guide, 2.9.2 is a well-tested choice.# Create a dedicated directory for Tika files
sudo mkdir -p /opt/tika

# Define the Tika version (recommended: latest stable 2.x)
TIKA_VERSION="2.9.2" # You can check https://tika.apache.org/download.html for the absolute latest stable 2.x release.

# Download the Tika server JAR
sudo wget "https://dlcdn.apache.org/tika/${TIKA_VERSION}/tika-server-standard-${TIKA_VERSION}.jar" -O /opt/tika/tika-server.jar

# Optional: Verify the integrity of the downloaded file using its SHA512 checksum.
# Find the official SHA512 hash on the Apache Tika download page (next to the JAR link).
# Then, compare it with the hash generated from your downloaded file:
# sha512sum /opt/tika/tika-server.jar
1.2 Create a systemd Service for TikaCreating a systemd service ensures Tika starts automatically on boot and can be easily managed.Note on User: During our troubleshooting, running Tika as the root user resolved startup issues specific to your LXC environment. While it's generally best practice to run services with the least possible privileges (e.g., as a dedicated tikauser), root is used here for proven reliability in your particular setup.sudo nano /etc/systemd/system/tika.service
Paste the following content into the file:[Unit]
Description=Apache Tika Server
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/java -jar /opt/tika/tika-server.jar --host 0.0.0.0 --port 9998
Restart=always
User=root           # Running as root for proven stability in this LXC setup
Group=root          # Running as root for proven stability in this LXC setup
WorkingDirectory=/opt/tika
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
Save and exit the editor: Press Ctrl+X, then Y to confirm saving, and Enter.1.3 Reload systemd, Enable, and Start Tika Service# Reload the systemd manager configuration
sudo systemctl daemon-reload

# Enable the Tika service to start on boot
sudo systemctl enable tika.service

# Start the Tika service immediately
sudo systemctl start tika.service
1.4 Verify Tika Server StatusConfirm that the Tika service is running and accessible on its expected port.# Check the service status
sudo systemctl status tika.service
The output should show Active: active (running).# Test connectivity to Tika's health endpoint
curl -X GET http://localhost:9998/tika
You should receive an XML response from the Tika server, indicating that it's active and listening for requests.2. Install Docker (for Gotenberg)Gotenberg is primarily distributed as a Docker container, as it efficiently bundles its complex dependencies (like LibreOffice and Chromium) into an isolated environment. We will install Docker within your LXC container specifically to run Gotenberg.# Update package list and install necessary prerequisites for Docker
sudo apt update
sudo apt install -y ca-certificates curl gnupg

# Create a directory for Docker's GPG key
sudo install -m 0755 -d /etc/apt/keyrings

# Download and add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the Docker repository to your Apt sources list
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Update Apt again to include the newly added Docker repository
sudo apt update

# Install the Docker packages
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Start and enable the Docker service to ensure it runs automatically:sudo systemctl start docker
sudo systemctl enable docker
Verify that Docker is running:sudo systemctl status docker
The output should indicate Active: active (running).3. Install and Configure GotenbergGotenberg is crucial for converting Office documents and emails into PDFs, which Paperless-ngx then processes.3.1 Run Gotenberg Docker ContainerWe will run the Gotenberg container, mapping its default port 3000 to the LXC host's port 3000.sudo docker run -d --restart unless-stopped --name gotenberg -p 3000:3000 gotenberg/gotenberg:7
gotenberg/gotenberg:7: Specifies Gotenberg version 7, which is recommended for compatibility with Paperless-ngx 2.16.3 to avoid breaking changes introduced in Gotenberg 8.x.3.2 Verify Gotenberg Container StatusCheck if the Gotenberg Docker container is successfully running and accessible.sudo docker ps -a | grep gotenberg
You should see a line for the gotenberg container with its STATUS showing Up ....curl http://localhost:3000/
You should receive a "Not Found" (HTTP 404) response. This is the expected behavior for Gotenberg's base URL and confirms it is running and listening for requests.4. Configure Paperless-ngxNow, instruct Paperless-ngx to utilize the running Tika and Gotenberg services by updating its main configuration file.Edit your paperless.conf file.Based on common bare-metal installations, this file is typically located at /opt/paperless/paperless.conf:sudo nano /opt/paperless/paperless.conf
Add or uncomment the following configuration lines.Ensure these variables are set correctly, uncommented (remove # from the beginning of the line if present):PAPERLESS_TIKA_ENABLED=true
PAPERLESS_TIKA_ENDPOINT=http://localhost:9998
PAPERLESS_TIKA_GOTENBERG_ENDPOINT=http://localhost:3000
Save and exit the editor (Ctrl+X, Y, Enter).5. Restart Paperless-ngx ServicesFor Paperless-ngx to load and apply the new Tika and Gotenberg configuration, you must restart its active services.sudo systemctl restart paperless-webserver.service
sudo systemctl restart paperless-task-queue.service
sudo systemctl restart paperless-consumer.service  # Include if this service exists in your setup
sudo systemctl restart paperless-scheduler.service # Include if this service exists in your setup
6. Final Verification and TestingCheck overall system service status:sudo systemctl status
Confirm that Tika (tika.service), Docker (docker.service), and all Paperless-ngx services (paperless-webserver.service, paperless-task-queue.service, etc.) are reporting as active (running). The system's degraded state (if any) should now be resolved.Upload and process documents through the Paperless-ngx web interface:Go to your Paperless-ngx web application.Upload a Microsoft Office document (e.g., a .docx file, a .xlsx spreadsheet, or a .pptx presentation).Upload an email file (e.g., an .eml file).Monitor the document consumption process within Paperless-ngx. If these documents are consumed successfully without "Connection refused" errors and their full text content becomes searchable within Paperless-ngx, then Tika and Gotenberg are fully integrated and working correctly!
