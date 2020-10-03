https://streamable.com/e/kq81lc

This is not safe to use as far as I know, virtually no testing done yet, just a reference for wrapping ideas

# Quick RTL UI + LND/NEUTRINO LOOP wrapper w/ nwjs for Windows

282 MB compressed zip file

786 MB uncompressed

no installation necessary, portable

Can either use this to run RTL.exe (pass `satoshi`) or assemble yourself from parts via instructions below

Zip file too big for github: https://drive.google.com/file/d/1a-VSgAWe61M3GRymp7YzXpTff4axPyUm/view?usp=sharing

It lets user set everything up through UI, initial default RTL pass `satoshi` was set that can be changed  in settings.

All the paths to macaroons and certs are automatically prepared by using relative paths to where lnd/loop is expected to generate them within the portable folder. I just wanted to skip of all the gotchas to test things, but figured should share.

I haven't had time yet to learn like anything about angular and what files I can get rid of so it's HUGE. ~300mb archive and I think I avoided dev dependencies but w/e - seems to simply work after unzipping, but more important is how anyone can do it:

# How to do it from open source codebases

All the dependencies are inside the folder & relatively pathed

Make folder anywhere like `RTL`.

Go inside and place https://github.com/Ride-The-Lightning/RTL inside app subfolder
by doing this:
```
git clone https://github.com/Ride-The-Lightning/RTL.git app
cd app
npm install --only=prod
```
(no rtl-config prep necessary - macroon file will be filled in based on path upon launch + known relative paths inside, it actually breaks if putting in relative paths into it manually when I tried so have to put in full paths from wrapper)
I preprogrammed RTL multiPass to be `satoshi` because it was too many passwords to enter at start while testing, is changed in settings easily.

From https://nwjs.io/ unzip v0.48.2 normal NW.js in main folder and I just renamed nwjs.exe to RTL.exe because it was confusing me

For binaries I have `RTL/bin/lnd_mainnet/` and `RTL/bin/lnd_testnet/` for lnd.exe and lncli.exe from https://github.com/LN-Zap/lnd/releases/tag/v0.11.0-beta-2-gfa327664d
Which is basically Zap Desktop's compiled binaries with all the flags on for simplicity of the work by https://github.com/lightningnetwork/lnd/

loop.exe and loopd.exe go from https://github.com/lightninglabs/loop/releases/tag/v0.8.1-beta go into  `RTL/bin/loop_mainnet/` and `RTL/bin/loop_testnet/`

to tell nwjs what to run for wrapper, have to make `RTL/package.json`
```
{
  "name": "rtl",
  "main": "./ui/ui.html",
  "node-main": "./app/rtl.js",
  "node-remote": "http://localhost:3000",
  "window": {
    "title": "RTL",
    "icon": "./app/src/assets/images/favicon-light/favicon-32x32.png",
    "frame": true,
    "position": "null",
    "resizable": true
  }
}
```

This handles the wrapper ui & built in scripts:

`RTL/ui/ui.html`
```
<!DOCTYPE html>
<html>
<head>
  <title>RTL</title>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link rel="stylesheet" href="ui.css">
</head>
<body>
  <iframe id="rtl" src="http://localhost:3000" onLoad="handleNav()"></iframe>
  <div id="infolog">debug logs</div>
  <script src="ui.js"></script>
</body>
</html>
```

`RTL/ui/ui.js`
```
const nwin = require('nw.gui').Window.get()
const { spawn, exec } = require('child_process')
const fs = require('fs')

const ROOTPATH = nw.App.startPath // nwjs RTL executable path
const RTL_SETTINGS_PATH = './app/RTL-Config.json'
const SETTINGS_JSON_PATH = './settings.json'

let WHICH_NETWORK = 'testnet'

// try to read settings otherwise assume testnet and new path
// if change in path, close app

let abort = false
let lastPath = ''
let isMainnet = false
try {
  const settings = JSON.parse(fs.readFileSync(SETTINGS_JSON_PATH, 'utf8'))
  lastPath = settings.lastPath
  isMainnet = settings.isMainnet
} catch (e) {}

try {
  // update settings path
  fs.writeFileSync(
    SETTINGS_JSON_PATH,
    JSON.stringify({ lastPath: ROOTPATH, isMainnet }, null, 2)
  )
  if (lastPath !== ROOTPATH) abort = true

  // update rtl config
  WHICH_NETWORK = isMainnet ? 'mainnet' : WHICH_NETWORK
  const LND_MACAROON_TESTNET = `\\bin\\lnd_testnet\\data\\chain\\bitcoin\\testnet`
  const LND_MACAROON_MAINNET = `\\bin\\lnd_mainnet\\data\\chain\\bitcoin\\mainnet`
  const LND_MACAROON = isMainnet ? LND_MACAROON_MAINNET : LND_MACAROON_TESTNET
  const rtlConfigText = fs.readFileSync(RTL_SETTINGS_PATH, 'utf8')
  log(rtlConfigText)
  const rtlConfig = JSON.parse(rtlConfigText)
  rtlConfig.nodes[0].Authentication.macaroonPath = ROOTPATH + LND_MACAROON
  rtlConfig.nodes[0].Settings.channelBackupPath =
    ROOTPATH + `\\backup\\${WHICH_NETWORK}`
  fs.writeFileSync(RTL_SETTINGS_PATH, JSON.stringify(rtlConfig, null, 2))
} catch (e) {
  log('pathcheck.json error:', e)
}
if (abort) {
  // run restarting via bat file after closing this app
  ;(async () => {
    const restartHelper = spawn('cmd.exe', ['/c', 'RESTART_APP.bat'], {
      detached: true,
      stdio: 'ignore'
    })
    restartHelper.unref()
    await new Promise(res => setTimeout(res, 1000))

    nwin.close()
  })()
}

// This runs lnd & loop on launch, kills on exit
const runBackgroundTasks = async () => {
  // running these batch files to run other ones
  // so large amount of text doesn't overflow buffer ever
  const lnd = exec(`LND${WHICH_NETWORK}.bat`, (err, stdout, stderr) => {
    if (err) {
      log('lnd error:', err)
      return
    }
    log('lnd:', stdout)
  })
  const loop = exec(`LOOP${WHICH_NETWORK}.bat`, (err, stdout, stderr) => {
    if (err) {
      log('loop error:', err)
      return
    }
    log('loop:', stdout)
  })

  // on window close destroy lnd and loop
  nwin.on('close', async () => {
    try {
      lnd.stdin.pause()
      spawn('taskkill', ['/pid', lnd.pid, '/f', '/t'])
    } catch (e) {
      log(e)
    }
    try {
      loop.stdin.pause()
      spawn('taskkill', ['/pid', loop.pid, '/f', '/t'])
    } catch (e) {
      log(e)
    }
    await new Promise(res => setTimeout(res, 2000))
const nwin = require('nw.gui').Window.get()
const { spawn, exec } = require('child_process')
const fs = require('fs')

const ROOTPATH = nw.App.startPath // nwjs RTL executable path
const RTL_SETTINGS_PATH = './app/RTL-Config.json'
const SETTINGS_JSON_PATH = './settings.json'

let WHICH_NETWORK = 'testnet'

// try to read settings otherwise assume testnet and new path
// if change in path, close app

let abort = false
let lastPath = ''
let isMainnet = false
try {
  const settings = JSON.parse(fs.readFileSync(SETTINGS_JSON_PATH, 'utf8'))
  lastPath = settings.lastPath
  isMainnet = settings.isMainnet
} catch (e) {
  abort = true
}

try {
  // update settings path
  fs.writeFileSync(
    SETTINGS_JSON_PATH,
    JSON.stringify({ lastPath: ROOTPATH, isMainnet }, null, 2)
  )
  if (lastPath !== ROOTPATH) abort = true

  // update rtl config
  WHICH_NETWORK = isMainnet ? 'mainnet' : WHICH_NETWORK
  const LND_MACAROON_TESTNET = `\\bin\\lnd_testnet\\data\\chain\\bitcoin\\testnet`
  const LND_MACAROON_MAINNET = `\\bin\\lnd_mainnet\\data\\chain\\bitcoin\\mainnet`
  const LND_MACAROON = isMainnet ? LND_MACAROON_MAINNET : LND_MACAROON_TESTNET
  const rtlConfigText = fs.readFileSync(RTL_SETTINGS_PATH, 'utf8')
  log(rtlConfigText)
  const rtlConfig = JSON.parse(rtlConfigText)
  rtlConfig.nodes[0].Authentication.macaroonPath = ROOTPATH + LND_MACAROON
  rtlConfig.nodes[0].Settings.channelBackupPath =
    ROOTPATH + `\\backup\\${WHICH_NETWORK}`
  fs.writeFileSync(RTL_SETTINGS_PATH, JSON.stringify(rtlConfig, null, 2))
} catch (e) {
  log('pathcheck.json error:', e)
}
if (abort) {
  // run restarting via bat file after closing this app
  ;(async () => {
    const restartHelper = spawn('cmd.exe', ['/c', 'RESTART_APP.bat'], {
      detached: true,
      stdio: 'ignore'
    })
    restartHelper.unref()
    await new Promise(res => setTimeout(res, 100))

    nwin.close()
  })()
}

// This runs lnd & loop on launch, kills on exit
const runBackgroundTasks = async () => {
  // running these batch files to run other ones
  // so large amount of text doesn't overflow buffer ever
  const lnd = exec(`LND${WHICH_NETWORK}.bat`, (err, stdout, stderr) => {
    if (err) {
      log('lnd error:', err)
      return
    }
    log('lnd:', stdout)
  })
  const loop = exec(`LOOP${WHICH_NETWORK}.bat`, (err, stdout, stderr) => {
    if (err) {
      log('loop error:', err)
      return
    }
    log('loop:', stdout)
  })

  // on window close destroy lnd and loop
  nwin.on('close', async () => {
    try {
      lnd.stdin.pause()
      spawn('taskkill', ['/pid', lnd.pid, '/f', '/t'])
    } catch (e) {
      log(e)
    }
    try {
      loop.stdin.pause()
      spawn('taskkill', ['/pid', loop.pid, '/f', '/t'])
    } catch (e) {
      log(e)
    }
    await new Promise(res => setTimeout(res, 500))
    nwin.close(true)
  })
  // bring to front, only this works
  nwin.setAlwaysOnTop(true)
  await new Promise(res => setTimeout(res, 500))
  nwin.setAlwaysOnTop(false)
}
runBackgroundTasks()

// This changes window title to current path inside the app
const handleNav = () => {
  document.title = `RTL ${
    document.getElementById('rtl').contentWindow.location.href
  }`
}

// Makes log window toggled by F11 key so log(strings) can go somewhere
function log (...text) {
  const infolog = document.getElementById('infolog')
  infolog.innerHTML += '\n' + text.map(v => v.toString()).join(' ')
}
log(`App.startPath: ${nw.App.startPath}`)
nw.App.registerGlobalHotKey(
  new nw.Shortcut({
    key: 'F11',
    active: () => {
      const rtl = document.getElementById('rtl')
      const infolog = document.getElementById('infolog')
      if (infolog.style.display !== 'block') {
        rtl.style.height = '80%'
        infolog.style.display = 'block'
      } else {
        rtl.style.height = '100%'
        infolog.style.display = 'none'
      }
    }
  })
)

// This on launch resizes window to 75% of screen
nwin.maximize()
const [w, h] = [Math.round(nwin.width * 0.75), Math.round(nwin.height * 0.75)]
nwin.restore()
nwin.resizeTo(w, h)
```

`RTL/ui/ui.css`
```
html,
body,
div,
iframe {
  display: block;
  box-sizing: border-box;
  margin: 0;
  padding: 0;
  width: 100%;
  height: 100%;
  -webkit-app-region: drag;
  border: 0;
  background-color: white;
}

iframe {
  width: 100%;
  height: 100%;
}

#infolog {
  display: none;
  border-top: 1px solid black;
  padding: 1%;
  width: 100%;
  height: 20%;
  white-space: pre-wrap;
  background-color: white;
  overflow-y: scroll;
  overflow-x: hidden;
}
```

BAT files to handle background lnd and loop processes.

these basically launch visible cmd windows so can see if frozen, pause helps send termination commands to all the spawns.

`LNDtestnet.bat` 
```
cd bin/lnd_testnet/
start cmd /k Call 1_lnd.bat
PAUSE
```

`LNDmainnet.bat`
```
cd bin/lnd_mainnet/
start cmd /k Call 1_lnd.bat
PAUSE
```

`LOOPtestnet.bat`
```
cd bin/loop_testnet/
start cmd /k Call 1_loopd.bat
PAUSE
```

`LOOPmainnet.bat`
```
cd bin/loop_mainnet/
start cmd /k Call 1_loopd.bat
PAUSE
```

`RESTART_APP.bat` - if RTL is missing config settings, this is spawned isolated to reopen the app with new settings applied after ~5 seconds
```
timeout /t 2 /nobreak
RTL.exe
SLEEP 1
```


backups will automaticlaly go to `RTL/backup/testnet` or `RTL/backup/mainnet` that can be changed from RTL settings

Ok this is getting very long

`RTL/bin/lnd_testnet/1_lnd.bat` just launches lnd with local config file (all data files go in a folder next to this file)
```
lnd --configfile lnd.conf
pause
```

`RTL/bin/lnd_testnet/lnd.conf`
```
[Application Options]

datadir=./data
adminmacaroonpath=./data/chain/bitcoin/testnet/admin.macaroon
readonlymacaroonpath=./data/chain/bitcoin/testnet/readonly.macaroon
invoicemacaroonpath=./data/chain/bitcoin/testnet/invoice.macaroon
tlscertpath=./data/tls.cert
tlskeypath=./data/tls.key

logdir=./logs
maxlogfiles=3
maxlogfilesize=10

listen=0.0.0.0:9735
rpclisten=0.0.0.0:11009
restlisten=0.0.0.0:8080
tlsextraip=0.0.0.0
tlsautorefresh=true

maxpendingchannels=50
minchansize=50000
stagger-initial-reconnect=true
max-channel-fee-allocation=1.0

accept-keysend=true
allow-circular-route=true

[Bitcoin]
bitcoin.active=true
bitcoin.testnet=true
bitcoin.node=neutrino
bitcoin.defaultchanconfs=1

[neutrino]
neutrino.addpeer=btcd-testnet.ion.radar.tech
neutrino.addpeer=btcd-testnet.lightning.computer
neutrino.addpeer=lnd.bitrefill.com:18333
neutrino.addpeer=faucet.lightning.community
neutrino.addpeer=testnet1-btcd.zaphq.io
neutrino.addpeer=testnet2-btcd.zaphq.io
neutrino.addpeer=testnet3-btcd.zaphq.io
neutrino.addpeer=testnet4-btcd.zaphq.io

neutrino.feeurl=https://nodes.lightning.computer/fees/v1/btctestnet-fee-estimates.json

[protocol]
protocol.wumbo-channels=true

[routing]
routing.assumechanvalid=true

[autopilot]
autopilot.active=false
autopilot.maxchannels=50
autopilot.allocation=1.0
autopilot.minchansize=200000
autopilot.maxchansize=16000000
autopilot.private=false
autopilot.minconfs=1
autopilot.conftarget=2

[routerrpc]
routerrpc.apriorihopprob=0.5
routerrpc.aprioriweight=0.75
routerrpc.attemptcost=10
routerrpc.maxmchistory=10000
routerrpc.minrtprob=0.005
routerrpc.penaltyhalflife=1h0m0s
```

loop does similar but as it has to wait for lnd it retries every N seconds (ui wrapper handles its termination)
`RTL/bin/loop_testnet/1_loopd.bat` (all data files go in a folder next to this file)
```
:loop
loopd --configfile loopd.conf
timeout /t 30 /nobreak
goto loop
```

`RTL/bin/loop_testnet/loopd.conf`
```
# loop
network=testnet
loopdir=./data
loopoutmaxparts=5
debuglevel=debug

# lnd
lnd.host=localhost:11009
lnd.macaroondir=./../lnd_testnet/data/chain/bitcoin/testnet
lnd.tlspath=./../lnd_testnet/data/tls.cert
```

`RTL/bin/lnd_mainnet/lnd.conf` just needs to have different peers for neutrino client
```
[Application Options]

datadir=./data
adminmacaroonpath=./data/chain/bitcoin/mainnet/admin.macaroon
readonlymacaroonpath=./data/chain/bitcoin/mainnet/readonly.macaroon
invoicemacaroonpath=./data/chain/bitcoin/mainnet/invoice.macaroon
tlscertpath=./data/tls.cert
tlskeypath=./data/tls.key

logdir=./logs
maxlogfiles=3
maxlogfilesize=10

listen=0.0.0.0:9735
rpclisten=0.0.0.0:11009
restlisten=0.0.0.0:8080
tlsextraip=0.0.0.0
tlsautorefresh=true

maxpendingchannels=50
minchansize=50000
stagger-initial-reconnect=true
max-channel-fee-allocation=1.0

accept-keysend=true
allow-circular-route=true

[Bitcoin]
bitcoin.active=true
bitcoin.mainnet=true
bitcoin.node=neutrino
bitcoin.defaultchanconfs=1

[neutrino]
neutrino.addpeer=btcd-mainnet.lightning.computer
neutrino.addpeer=mainnet1-btcd.zaphq.io
neutrino.addpeer=mainnet2-btcd.zaphq.io
neutrino.addpeer=mainnet3-btcd.zaphq.io
neutrino.addpeer=mainnet4-btcd.zaphq.io
neutrino.addpeer=btcd0.lightning.engineering

neutrino.feeurl=https://nodes.lightning.computer/fees/v1/btc-fee-estimates.json

[protocol]
protocol.wumbo-channels=1

[routing]
routing.assumechanvalid=1

[autopilot]
autopilot.active=false
autopilot.maxchannels=50
autopilot.allocation=1.0
autopilot.minchansize=200000
autopilot.maxchansize=16000000
autopilot.private=false
autopilot.minconfs=1
autopilot.conftarget=2

[routerrpc]
routerrpc.apriorihopprob=0.5
routerrpc.aprioriweight=0.75
routerrpc.attemptcost=10
routerrpc.maxmchistory=10000
routerrpc.minrtprob=0.005
routerrpc.penaltyhalflife=1h0m0s
```

Switching between mainnet or testnet can be done by editing "isMainnet" entry to true or false inside

`settings.json`
```
{
  "lastPath": "SOME_PATH",
  "isMainnet": false
}
```

and basically same thing for w/e other network

Now my goals are to improve on its security and reduce its size if possible.

2020 09 29 - updated the lnd.conf and edited RTL `./controllers/lnd/wallet.js` to add `recovery_window: 20000` for `initwallet` initialized wallets and `unlockwallet` because it wasn't scanning them - issue to address this has been created at RTL github (num 494)

2020 10 03 - removed alias from default lnd conf, added video, used higher compression for zip file but still big: 282 MB
