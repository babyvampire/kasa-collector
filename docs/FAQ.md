# FAQ (Frequently Asked Questions)

## Discovery and Network Issues

### Q: Does the collector need to be on the same network/VLAN as my Kasa devices?

**Short answer**: For auto-discovery yes, but manual configuration works across VLANs with proper routing.

**Detailed answer**: 

The collector uses **UDP broadcast packets on port 9999** to discover devices:

1. Collector sends UDP broadcast to `255.255.255.255:9999`
2. Kasa devices receive the broadcast and respond back with their information
3. Collector listens for these UDP responses to build its device list

This is **not mDNS** - it's TP-Link's proprietary discovery protocol that requires bidirectional UDP communication.

### Q: My collector is on a different VLAN than my Kasa devices. Why doesn't discovery work?

Common VLAN setups like this will break discovery:
- ✅ Server VLAN → IoT VLAN (collector can send discovery packets)
- ❌ IoT VLAN → Server VLAN (devices can't send responses back)

The discovery breaks because while your collector can send the discovery broadcast, the Kasa devices' responses are blocked by your firewall rules.

**Solutions:**

1. **Manual Configuration (Recommended)**
   ```yaml
   environment:
     KASA_COLLECTOR_ENABLE_AUTO_DISCOVERY: "false"
     KASA_COLLECTOR_DEVICE_HOSTS: "192.168.2.100,192.168.2.101,192.168.2.102"
   ```
   This works perfectly with one-way communication because:
   - No UDP discovery needed
   - Collector connects directly via TCP port 9999
   - Only requires Server → IoT communication

2. **Limited Firewall Exception**
   Add a minimal rule to allow discovery responses:
   - From: IoT VLAN
   - To: Server VLAN (specifically the collector IP)
   - Port: UDP 9999 (source port)
   - Direction: Return traffic only

3. **Deploy on IoT VLAN**
   Run the collector on the same VLAN as your Kasa devices.

### Q: What ports does Kasa Collector use?

- **UDP 9999**: Device discovery (broadcast)
- **TCP 9999**: Device communication (IOT devices)
- **TCP 80/443**: Device communication (SMART devices)

### Q: Can I use Kasa Collector without auto-discovery?

Yes! Set `KASA_COLLECTOR_ENABLE_AUTO_DISCOVERY=false` and provide device IPs via `KASA_COLLECTOR_DEVICE_HOSTS`. This is actually more secure and works across VLANs.

## Authentication Issues

### Q: I'm getting "Server response doesn't match our challenge" errors

This means your devices require TP-Link cloud credentials. Add these environment variables:
```yaml
KASA_COLLECTOR_TPLINK_USERNAME: your-email@example.com
KASA_COLLECTOR_TPLINK_PASSWORD: your-password
```

### Q: Which devices need authentication?

Newer devices (firmware from ~2021 onwards) like KP125M, EP25, and some HS103 units require authentication. Older devices work without credentials.

## Performance and Data Collection

### Q: How often does the collector poll devices?

- Energy data (power, current, voltage): Every 15 seconds (configurable via `KASA_COLLECTOR_DATA_FETCH_INTERVAL`)
- System info (state, wifi signal): Every 60 seconds (configurable via `KASA_COLLECTOR_SYSINFO_FETCH_INTERVAL`)

### Q: Why is my InfluxDB database growing quickly?

With default settings, each device generates:
- 4 data points every 15 seconds (240 per minute)
- 1 data point every 60 seconds (1 per minute)
- Total: ~241 points per minute per device

For 10 devices, that's ~3.5 million data points per day. Consider:
- Increasing fetch intervals
- Configuring InfluxDB retention policies
- Using InfluxDB downsampling

### Q: Can I reduce logging verbosity?

Yes! The collector intelligently reduces logs after the first discovery. You can also set:
```yaml
KASA_COLLECTOR_LOG_LEVEL_KASA_COLLECTOR: WARNING
KASA_COLLECTOR_LOG_LEVEL_KASA_API: WARNING
```

## Docker and Deployment

### Q: Why does the container need `network_mode: host`?

Host networking is required for UDP broadcast discovery. If using manual device configuration, you could use bridge networking with proper port mapping.

### Q: How do I check if the collector is healthy?

The collector includes a built-in health check:
```bash
docker inspect kasa-collector | jq '.[0].State.Health'
```

It monitors data file freshness - if no data is written for 2 minutes (configurable), it reports unhealthy.

## Device Compatibility

### Q: My device is discovered but won't connect

Common causes:
1. Device needs power cycle
2. Different VLAN without proper routing
3. Firewall blocking TCP port 9999
4. Device firmware issue

Try:
- Power cycling the device
- Using manual configuration with the device IP
- Checking firewall logs

### Q: Are all Kasa devices supported?

The collector works with most Kasa smart plugs and power strips that support energy monitoring. Devices without energy monitoring (like switches) will have limited data collection.