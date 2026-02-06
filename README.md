# Keycloak IntelliJ Remote Debugging Setup

This guide walks you through building a custom Keycloak Docker image with your code changes and setting up remote debugging with IntelliJ IDEA.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Steps](#steps)
- [Troubleshooting](#troubleshooting)
- [Quick Reference](#quick-reference)
- [Contributing](#contributing)
- [Issues and Discussions](#issues-and-discussions)
- [License](#license)

## Prerequisites

- Docker installed and running
- Kubernetes cluster (minikube, kind, or any k8s cluster)
- kubectl configured
- IntelliJ IDEA
- Maven

## Steps

### 1. Compile Keycloak with Your Changes

From the root of the Keycloak repository:

```bash
./mvnw clean install -DskipTests -Pdistribution
```

This builds the distribution without running tests to save time.

### 2. Check the Built Version

Navigate to the keycloak directory and verify the version:

```bash
cd keycloak/ #root repo
cat pom.xml | grep -A 1 "<artifactId>keycloak-parent</artifactId>" | grep version
./mvnw help:evaluate -Dexpression=project.version -q -DforceStdout
```

**Sample output:**
```
<version>999.0.0-SNAPSHOT</version>
999.0.0-SNAPSHOT
```

Note this version number - you'll need it for the Docker build.

### 3. Build Docker Image

Navigate to the container directory and build the Docker image:

```bash
cd ./quarkus/container
```
```bash
cp ../dist/target/keycloak-999.0.0-SNAPSHOT.tar.gz .
```
Build with the version from step 2:

```bash
docker build \
  --build-arg KEYCLOAK_DIST=keycloak-999.0.0-SNAPSHOT.tar.gz \
  --build-arg KEYCLOAK_VERSION=999.0.0-SNAPSHOT \
  -t my-keycloak:debug \
  -f Dockerfile \
  .
```

**Note:** Replace `999.0.0-SNAPSHOT` with your actual version if different.

### 4. Deploy to Kubernetes

Apply the Keycloak debug configuration:

```bash
kubectl apply -f keycloak-debug.yaml
```

Wait for the pod to be ready:

```bash
kubectl get pods -w
```

### 5. Set Up Port Forwarding

Forward both the debug port and the application port:

```bash
# Debug port (for IntelliJ remote debugging)
kubectl port-forward service/keycloak-debug 5005:5005

# Application port (for accessing Keycloak UI)
kubectl port-forward service/keycloak-debug 8080:8080
```

**Note:** Run these in separate terminal windows/tabs, or run them in the background.

To run in background:

```bash
kubectl port-forward service/keycloak-debug 5005:5005 &
kubectl port-forward service/keycloak-debug 8080:8080 &
```

### 6. Configure IntelliJ IDEA Remote Debugging

1. Open your Keycloak project in IntelliJ IDEA
2. Go to **Run** → **Edit Configurations**
3. Click **+** → **Remote JVM Debug**
4. Configure:
   - **Name:** Keycloak Debug
   - **Host:** localhost
   - **Port:** 5005
   - **Debugger mode:** Attach to remote JVM
   - **Use module classpath:** keycloak-quarkus-server
5. Click **OK**

### 7. Start Debugging

1. Set breakpoints in your code where needed
2. Click the **Debug** button or select **Run** → **Debug 'Keycloak Debug'**
3. Access Keycloak at `http://localhost:8080`
4. Your breakpoints should now trigger when the code is executed

## Troubleshooting

### Port Forwarding Issues

If port forwarding stops working or you get "port already in use" errors:

**Find and kill existing port-forward processes:**

```bash
# Kill all kubectl port-forward processes
pkill -f "kubectl port-forward"

# Or kill specific port
lsof -ti:5005 | xargs kill -9
lsof -ti:8080 | xargs kill -9
```

Then restart the port-forward commands.

### Pod Not Starting

Check pod logs:

```bash
kubectl logs -f deployment/keycloak-debug
```

Check pod status:

```bash
kubectl describe pod <pod-name>
```

### IntelliJ Can't Connect

1. Verify port-forward is running: `lsof -i:5005`
2. Check that the pod has debug port exposed in `keycloak-debug.yaml`
3. Ensure firewall isn't blocking port 5005
4. Restart IntelliJ debugger

### Rebuilding After Code Changes

After making code changes:

1. Stop the debugger in IntelliJ
2. Delete the Kubernetes deployment: `kubectl delete -f keycloak-debug.yaml`
3. Repeat steps 1-7 above

## Quick Reference

**Kill all port forwards:**
```bash
pkill -f "kubectl port-forward"
```

**Restart deployment:**
```bash
kubectl rollout restart deployment/keycloak-debug
```

**View logs:**
```bash
kubectl logs -f deployment/keycloak-debug
```

**Access Keycloak:**
- **UI:** http://localhost:8080
- **Debug Port:** localhost:5005

## Notes

- The `keycloak-debug.yaml` should have debug configuration enabled (JAVA_OPTS with debug parameters)
- Default Keycloak admin credentials are typically set in the YAML file
- Port 5005 is the standard Java debug port
- Port 8080 is the default Keycloak HTTP port

## Contributing

Contributions are welcome! If you have improvements or suggestions for this debugging setup:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -s -m 'Add some improvement'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Open a Pull Request

Please ensure your commits are signed off using the `-s` flag to indicate you agree to the Developer Certificate of Origin.

## Issues and Discussions

- **Found a bug?** Open an [issue](../../issues) with detailed steps to reproduce
- **Want to suggest an enhancement?** Open an [issue](../../issues) with the enhancement label or start a discussion

When reporting issues, please include:
- Your environment (OS, Docker version, Kubernetes version)
- Keycloak version you're building
- Complete error messages and logs
- Steps to reproduce the problem

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

```
Copyright 2026 Sabeer

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```

---

## Disclaimer

This repository provides debugging setup instructions for [Keycloak](https://github.com/keycloak/keycloak), 
an open-source Identity and Access Management solution licensed under Apache License 2.0.

This repository does NOT contain Keycloak source code. Users must clone and build Keycloak from 
the official repository following the instructions provided here.
