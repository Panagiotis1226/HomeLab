# Vanilla Minecraft Server

## Prerequisites
- Minecraft Server Jar
- Java JDK 17+

## Steps

### 1. Download the Minecraft Server Jar to your directory
Download the Minecraft Server Jar from the official website or from a trusted source.

### 2. Run the Minecraft Server Jar

#### 2.1 Create a script to run the Minecraft Server Jar.
For Linux/MacOS:
```bash
nano run_server.sh
```

For Windows:
Create a txt file and name it `run_server.bat`.

Add the following to the script:

```bash
java -Xmx1024M -Xms1024M -jar {server_jar} nogui
```

You can modify the `-Xmx` (Max Memory) and `-Xms` (Initial Memory) to your desired memory allocation.

### 3. Start the Minecraft Server

#### 3.1 Run the script to start the Minecraft Server.
For Linux/MacOS:
```bash
./run_server.sh
```

For Windows:
```bash
run_server.bat
```
#### 3.2 Accept the EULA
You need to accept the EULA by creating or modifying the `eula.txt` file in the same directory as the server jar.

```bash
nano eula.txt
```

Add the following to the file:

```bash
eula=true
```

#### 3.3 Now re-run the script to start the Minecraft Server.

