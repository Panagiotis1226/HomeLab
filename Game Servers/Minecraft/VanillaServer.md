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

### 3. Configure the Minecraft Server

### 4. Start the Minecraft Server