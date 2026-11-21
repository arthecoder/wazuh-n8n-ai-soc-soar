#!/usr/bin/env python3
import sys
import json
import urllib.request

alert_file = open(sys.argv[1])
hook_url = sys.argv[3]
alert_json = json.loads(alert_file.read())
alert_file.close()

req = urllib.request.Request(
    hook_url,
    data=json.dumps(alert_json).encode('utf-8'),
    headers={'Content-Type': 'application/json', 'User-Agent': 'Wazuh-n8n-Integration'}
)
urllib.request.urlopen(req, timeout=10)
sys.exit(0)
