<script>
  let status = 'idle';
  let device = null;
  let server = null;
  let error = '';

  let services = [];

  // Hardcoded service and characteristic UUIDs from your device
  const HARD_SERVICE_UUID = 'f7547938-68ba-11ec-90d6-0242ac120003';
  const HARD_CHAR_UUIDS = [
    '9c85a726-b7f1-11ec-b909-0242ac120002',
    '12345678-1234-5678-1234-56789abcdef3'
  ];

  

  function bufferToHex(buf) {
    const view = new DataView(buf.buffer || buf);
    const parts = [];
    for (let i = 0; i < view.byteLength; i++) parts.push(view.getUint8(i).toString(16).padStart(2, '0'));
    return parts.join(' ');
  }

  function textFromDataView(view) {
    try {
      const decoder = new TextDecoder();
      return decoder.decode(view);
    } catch (e) {
      return bufferToHex(view);
    }
  }

  async function connect() {
    error = '';
    if (!navigator.bluetooth) {
      status = 'unsupported';
      console.log('unsoported');
      return;
    }

    status = 'requesting';
    try {
      // Use the hardcoded service UUID when requesting the device so we can access its characteristics
      const optionalServices = [HARD_SERVICE_UUID];
      const d = await navigator.bluetooth.requestDevice({
        filters: [{ namePrefix: 'Zephyr Sample' }],
        optionalServices
      });
      device = d;
      status = 'connecting';
      device.addEventListener && device.addEventListener('gattserverdisconnected', onDisconnected);
      if (device.gatt) {
        server = await device.gatt.connect();
      } else {
        server = null;
      }
      status = server && server.connected ? 'connected' : 'connected (no GATT)';

      if (server && server.connected) {
        // Discover only the hardcoded service and characteristics
        await discoverHardcoded();
      }
    } catch (err) {
      error = err && err.message ? err.message : String(err);
      status = 'idle';
    }
  }

  async function discoverHardcoded() {
    services = [];
    if (!server) return;
    try {
      const s = await server.getPrimaryService(HARD_SERVICE_UUID);
      const svc = { uuid: s.uuid, characteristics: [] };
      for (const cu of HARD_CHAR_UUIDS) {
        try {
          const c = await s.getCharacteristic(cu);
          const props = Object.keys(c.properties).filter(k => c.properties[k]);
          svc.characteristics.push({
            uuid: c.uuid,
            properties: props,
            characteristic: c,
            value: null,
            notifying: false
          });
        } catch (e) {
          // characteristic not found or other error; include as placeholder
          svc.characteristics.push({ uuid: cu, properties: [], characteristic: null, value: null, notifying: false, error: e && e.message });
        }
      }
      services.push(svc);
    } catch (e) {
      error = e && e.message ? e.message : String(e);
    }
  }

  async function discoverServices() {
    services = [];
    if (!server) return;
    try {
      const svcs = await server.getPrimaryServices();
      for (const s of svcs) {
        const svc = { uuid: s.uuid, characteristics: [] };
        try {
          const chars = await s.getCharacteristics();
          for (const c of chars) {
            const props = Object.keys(c.properties).filter(k => c.properties[k]);
            svc.characteristics.push({
              uuid: c.uuid,
              properties: props,
              characteristic: c,
              value: null,
              notifying: false
            });
          }
        } catch (e) {
          // ignore per-service characteristic errors
        }
        services.push(svc);
      }
    } catch (e) {
      error = e && e.message ? e.message : String(e);
    }
  }

  async function readCharacteristic(item) {
    try {
      // If characteristic object missing, try to (re)fetch it from the server/service
      if (!item.characteristic) {
        if (!server) throw new Error('Not connected to device');
        try {
          const svc = await server.getPrimaryService(HARD_SERVICE_UUID);
          const c = await svc.getCharacteristic(item.uuid);
          const props = Object.keys(c.properties).filter(k => c.properties[k]);
          item.characteristic = c;
          item.properties = props;
        } catch (inner) {
          item.error = inner && inner.message ? inner.message : String(inner);
          throw inner;
        }
      }

      const val = await item.characteristic.readValue();
      item.value = textFromDataView(val);
      console.log('Read value for characteristic', item.uuid, ':', val);
      item.error = null;
    } catch (e) {
      error = e && e.message ? e.message : String(e);
    }
  }

  async function readLEDState() {
    error = '';
    try {
      if (!server || !server.connected) throw new Error('Not connected to device');
      const svc = await server.getPrimaryService(HARD_SERVICE_UUID);
      const characteristic = await svc.getCharacteristic(HARD_CHAR_UUIDS[0]);
      const val = await characteristic.readValue();
      console.log('LED State value:', val);
      error = `LED State: ${textFromDataView(val)}`;
    } catch (e) {
      error = e && e.message ? e.message : String(e);
    }
  }

  async function readSecondCharacteristic() {
    error = '';
    try {
      if (!server || !server.connected) throw new Error('Not connected to device');
      const svc = await server.getPrimaryService(HARD_SERVICE_UUID);
      const characteristic = await svc.getCharacteristic(HARD_CHAR_UUIDS[1]);
      const val = await characteristic.readValue();
      console.log('Second characteristic value:', val);
      error = `Characteristic 2: ${textFromDataView(val)}`;
    } catch (e) {
      error = e && e.message ? e.message : String(e);
    }
  }

  async function toggleNotifications(item) {
    try {
      if (item.notifying) {
        await item.characteristic.stopNotifications();
        item.characteristic.removeEventListener('characteristicvaluechanged', item._listener);
        item.notifying = false;
      } else {
        const listener = (ev) => {
          const v = ev.target.value;
          item.value = textFromDataView(v);
        };
        item._listener = listener;
        await item.characteristic.startNotifications();
        item.characteristic.addEventListener('characteristicvaluechanged', listener);
        item.notifying = true;
      }
    } catch (e) {
      error = e && e.message ? e.message : String(e);
    }
  }

  function disconnect() {
    if (device && device.gatt && device.gatt.connected) {
      device.gatt.disconnect();
    }
    status = 'idle';
    device = null;
    server = null;
    services = [];
  }

  function onDisconnected() {
    status = 'disconnected';
    server = null;
    services = [];
  }
</script>

<main class="container">
  <h1>Track Rat — Bluetooth Connect</h1>
  <div class="layout">
    <div class="sidebar">
      <div class="controls">
        {#if status === 'connected' || status === 'connected (no GATT)'}
          <button class="btn btn-danger" on:click={disconnect}>Disconnect</button>
        {:else}
          <button class="btn btn-primary" on:click={connect}>Connect</button>
        {/if}
      </div>

      <div class="button-group">
        <button class="btn" on:click={readLEDState} disabled={status !== 'connected' && status !== 'connected (no GATT)'}>Read LED State</button>
        <button class="btn" on:click={readSecondCharacteristic} disabled={status !== 'connected' && status !== 'connected (no GATT)'}>Read Characteristic 2</button>
      </div>
    </div>

    <div class="content">
      <div class="card">
        <p class="status"><strong>Status:</strong> {status}</p>

        {#if device}
          <p><strong>Device:</strong> {device.name ?? '(unnamed)'} <small>({device.id})</small></p>
        {/if}

        <!-- Debug: show discovered services object for troubleshooting -->
        {#if status === 'connected'}
          <details style="margin-top:8px">
            <summary style="cursor:pointer">Show services debug</summary>
            <pre style="max-height:240px;overflow:auto;background:#f8fafc;padding:8px;border-radius:6px;border:1px solid #e6eefc">{JSON.stringify(services, null, 2)}</pre>
          </details>
        {/if}

        {#if error}
          <p class="error">{error}</p>
        {/if}

        {#if services.length}
          <section class="services">
            <h2>GATT Services</h2>
            {#each services as svc}
              <div class="service">
                <h3>Service: {svc.uuid}</h3>
                {#if svc.characteristics.length === 0}
                  <p class="muted">No discoverable characteristics</p>
                {:else}
                  <ul class="chars">
                    {#each svc.characteristics as ch}
                      <li class="char">
                        <div class="char-header">
                          <div class="char-id"><strong>{ch.uuid}</strong></div>
                          <div class="char-actions">
                            <!-- Always show Read Value button. If characteristic is missing, try to fetch it on demand. -->
                            <button class="btn" on:click={() => readCharacteristic(ch)}>Read Value</button>
                            {#if ch.properties.includes('notify')}
                              <button class="btn" on:click={() => toggleNotifications(ch)}>{ch.notifying ? 'Stop' : 'Notify'}</button>
                            {/if}
                          </div>
                        </div>
                        <div class="char-meta">Properties: {ch.properties.join(', ')}</div>
                        {#if ch.value}
                          <div class="char-value">Value: <code>{ch.value}</code></div>
                        {/if}
                        {#if ch.error}
                          <div class="char-error">Error: {ch.error}</div>
                        {/if}
                      </li>
                    {/each}
                  </ul>
                {/if}
              </div>
            {/each}
          </section>
        {/if}

        <p class="hint">Note: Web Bluetooth requires a secure context (HTTPS or localhost) and a supported browser (Chromium-based).</p>
      </div>
    </div>
  </div>
</main>
