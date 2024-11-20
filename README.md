npm run client (in root)
npm start (server folder)

# Sutaj Drinor & Sherifi Luan Docs
**1. Is it possible to deploy a Peer-to-Peer version of Mr. Wei implementation?**

Mr. Wei's implementation of the Differential Synchronization Algorithm is built  on a client-server architecture using WebSockets for real-time  communication. Deploying a Peer-to-Peer (P2P) version would require  significant modifications because the current setup relies on a central  server to coordinate state and handle synchronization between clients.

However, the core algorithm can be adapted for P2P communication. You would need to replace the central server with a P2P communication layer, such as  WebRTC, which allows direct communication between browsers. This  involves:

- **Peer Discovery**: Implementing a mechanism for peers to find and connect with each other.
- **Connection Management**: Handling the establishment and maintenance of P2P connections.
- **Message Passing**: Modifying the message handling to route synchronization messages between peers instead of through a server.

**2. How is it possible to use the API in other JS projects?**

Mr. Wei's code exports a `DiffSyncAlghorithm` class, which you can import and use in your own JavaScript projects. Here's how you can integrate it:

- **Installation**: If the package is published on npm, you can install it using `npm install diff-sync-js`. Otherwise, you can include the source code directly in your project.
- **Usage**: Import and initialize the `DiffSyncAlghorithm` class in your code.

```javascript

const DiffSyncAlgorithm = require('diff-sync-js'); // Adjust the path or package name as necessary
const jsonpatch = require('fast-json-patch'); // Ensure you have this dependency

const diffSync = new DiffSyncAlgorithm({
    jsonpatch: jsonpatch,
    thisVersion: 'n',
    senderVersion: 'm',
    useBackup: true,
    debug: true
});

// Initialize the container and main text
let container = {};
let mainText = ''; // Your initial document state
diffSync.initObject(container, mainText);

// Example of sending patches
function sendPatch(newText) {
    diffSync.onSend({
        container,
        mainText: newText,
        whenSend: (m, edits) => {
            // Send the patch to the other party
            sendMessage({
                type: 'PATCH',
                payload: {
                    m,
                    edits
                }
            });
        },
        whenUnchange: (m) => {
            // Optionally handle cases where there is no change
        }
    });
}

// Example of receiving patches
function receivePatch(payload) {
    diffSync.onReceive({
        payload,
        container,
        onUpdateShadow: (shadow, patch) => {
            // Update the shadow document
            return diffSync.strPatch(shadow.value, patch);
        },
        onUpdateMain: (patches, operations) => {
            // Apply the changes to your main document
            mainText = diffSync.strPatch(mainText, operations);
        },
        afterUpdate: () => {
            // Optionally send acknowledgments or further patches
        }
    });
}
```

**3. Are the JSON documents interchangeable with other kinds of documents?**

**4. How is Mr. Wei solving the conflicts?**

Mr. Wei's implementation handles conflicts using a combination of shadow copies  and backup copies of the documents. When a client or server receives a  patch, it checks version numbers to detect conflicts. If a conflict is  detected (e.g., version numbers don't match), it attempts to resolve it  by:

- **Reverting to a Backup**: If the shadow's  version doesn't match the received version but the backup's version  does, it reverts the shadow to the backup state.
- **Discarding Changes**: If neither the shadow nor the backup versions match, it may discard the changes to prevent inconsistent states.

Here's a code snippet illustrating this from `src/index.js`:

```

if (useBackup) {
    if (shadow[thisVersion] !== payload[thisVersion]) {
        // Versions don't match
        if (backup[thisVersion] === payload[thisVersion]) {
            // Revert to backup
            shadow.value = jsonpatch.deepClone(backup.value);
            shadow[thisVersion] = backup[thisVersion];
            shadow.edits = [];
        } else {
            // Cannot resolve conflict, drop the process
            return;
        }
    }
}
```

This mechanism ensures that both parties stay synchronized by resolving conflicts based on version control.





# diff-sync-js

A JavaScript implementation of Neil Fraser Differential Synchronization Algorithm

Diff Sync Writing: https://neil.fraser.name/writing/sync/

## Use Case

Differential synchronization algorithm keep two or more copies of the same document synchronized with each other in real-time. The algorithm offers scalability, fault-tolerance, and responsive collaborative editing across an unreliable network.

## Demo

http://wztechs.com/diff-sync-text-editor/demo/client/

## How to install

When use npm
`npm install diff-sync-js`

When use html
`<script src="./dist/diffSync.js"></script>`

## Dependencies

`json-fast-patch`

## To Test

`npm run test`

## How to use

1. Initial diffSync instance

```
    var diffSync = new DiffSyncAlghorithm({
        jsonpatch: jsonpatch,
        thisVersion: "m",
        senderVersion: "n",
        useBackup: true,
        debug: true
    });
```

2. Initialize container

```
    var container = {};
    diffSync.initObject(container, mainText);
```

3. When Send Payload

```
    diffSync.onSend({
        container,
        mainText,
        whenSend(senderVersion, edits) {
            send({
                type: "PATCH",
                {
                    senderVersion,
                    edits
                }
            });
        }
    });
```

4. When Receive Payload

```
    diffSync.onReceive({
        payload,
        container,
        onUpdateMain(patches, operations) {
            mainText = jsonpatch.applyPatch(mainText, operations).newDocument;
        },
        afterUpdate(senderVersion) {
            send({
                type: "ACK",
                payload: {
                    senderVersion
                }
            });
        }
    });
```

5. When Receive Ack

```
    diffSync.onAck(container, payload);
```

## API

constructor

```
     /**
     * @param {object} options.jsonpatch json-fast-patch library instance (REQUIRED)
     * @param {string} options.thisVersion version tag of the receiving end
     * @param {string} options.senderVersion version tag of the sending end
     * @param {boolean} options.useBackup indicate if use backup copy (DEFAULT true)
     * @param {boolean} options.debug indicate if print out debug message (DEFAULT false)
     */
    constructor({ jsonpatch, thisVersion, senderVersion, useBackup = true, debug = false })

```

initObject

```
    /**
     * Initialize the container
     * @param {object} container any
     * @param {object} mainText any
     */
    initObject(container, mainText)
```

onReceive

```
    /**
     * On Receive Packet
     * @param {object} options.payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     * @param {object} options.container container object {shadow, backup}
     * @param {func} options.onUpdateMain (patches, patchOperations, shadow[thisVersion]) => void
     * @param {func} options.afterUpdate (shadow[senderVersion]) => void
     * @param {func} options.onUpdateShadow (shadow, patch) => newShadowValue
     */
    onReceive({ payload, container, onUpdateMain, afterUpdate, onUpdateShadow })
```

onSend

```
    /**
     * On Sending Packet
     * @param {object} options.container container object {shadow, backup}
     * @param {object} options.mainText any
     * @param {func} options.whenSend (shadow[senderVersion], shadow.edits) => void
     * @param {func} options.whenUnchange (shadow[senderVersion]) => void
     */
    onSend({ container, mainText, whenSend, whenUnchange })

```

onAck

```
     /**
     * Acknowledge the other side when no change were made
     * @param {object} container container object {shadow, backup}
     * @param {object} payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     */
    onAck(container, payload)
```

onAck

```
    /**
     * Acknowledge the other side when no change were made
     * @param container container object {shadow, backup}
     * @param payload payload object that contains {thisVersion, edits}. Edits should be a list of {senderVersion, patch}
     */
    onAck(container, payload)
```

clearOldEdits

```
    /**
     * clear old edits
     * @param {object} shadow
     * @param {string} version
     */
    clearOldEdits(shadow, version)
```

strPatch

```
    /**
     * apply patch to string
     * @param {string} val
     * @param {patch} patch
     * @return string
     */
    strPatch(val, patch)
```
