# Beautiful Architecture

Hugo source for the `beautifularchitecture.github.io` GitHub Pages site.

## Access the development server from your local network

On Linux, run Hugo bound to all network interfaces and use `hostname -I` to select the computer's first reported IP address:

```sh
LAN_IP=$(hostname -I | awk '{print $1}')
echo "Open http://$LAN_IP:1313 on another device"
hugo server --bind 0.0.0.0 --baseURL "http://$LAN_IP:1313" --appendPort=false
```

Run the command from the repository root. Connect from another device on the same network using the URL printed by `echo`. If the computer has multiple network interfaces, check that the selected address is reachable from the other device. The `npm run dev:lan` shortcut uses the IP address configured in `package.json` rather than discovering one dynamically.

## GitHub Pages 404 Error

### Causes

- Exact Dates repeat
