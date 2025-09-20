<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Flight Operation Management System — Full Rebuild</title>
  <style>
    :root{--blue:#0b66c3;--muted:#6b7280;--card:#ffffff;--bg:#f7fbff}
    *{box-sizing:border-box}
    body{font-family:Inter,system-ui,Arial,Helvetica,sans-serif;margin:0;background:var(--bg);color:#0f172a}
    header{background:linear-gradient(90deg,var(--blue),#024a86);color:white;padding:16px}
    header h1{margin:0;font-size:20px}
    .container{max-width:1200px;margin:18px auto;padding:0 16px}
    .grid{display:grid;grid-template-columns:1fr 420px;gap:18px}
    .card{background:var(--card);border-radius:12px;padding:16px;box-shadow:0 6px 20px rgba(2,6,23,0.06)}
    .search-row{display:flex;gap:10px;flex-wrap:wrap}
    label{font-size:13px;color:var(--muted)}
    input[type=text],input[type=date],input[type=number],input[type=email],select{padding:10px;border-radius:8px;border:1px solid #e6eef6;min-width:120px}
    button.btn{background:var(--blue);color:white;padding:8px 12px;border-radius:8px;border:0;cursor:pointer}
    button.ghost{background:#fff;border:1px solid #e6eef6;color:var(--blue);padding:6px 10px;border-radius:8px;cursor:pointer}
    .results{margin-top:12px}
    .flight-item{display:flex;justify-content:space-between;align-items:center;padding:12px;border-radius:10px;border:1px solid #edf2f7;margin-bottom:10px}
    .meta{font-size:13px;color:var(--muted)}
    .right-col{display:flex;flex-direction:column;gap:12px}
    .bookings-list{max-height:360px;overflow:auto}
    .passenger-entry{display:flex;justify-content:space-between;align-items:center;margin-bottom:6px}
    .muted{color:var(--muted);font-size:13px}
    .modal{position:fixed;inset:0;display:none;align-items:center;justify-content:center;background:rgba(2,6,23,0.45);z-index:40}
    .modal .panel{background:white;border-radius:10px;padding:18px;max-width:900px;width:96%;max-height:90vh;overflow:auto}
    table{width:100%;border-collapse:collapse}
    th,td{padding:8px;border-bottom:1px solid #eef2f7;text-align:left}
    footer{max-width:1200px;margin:18px auto;padding:0 16px 30px;color:var(--muted);font-size:13px}
    @media (max-width:980px){.grid{grid-template-columns:1fr}.right-col{order:2}}
  </style>
</head>
<body>
<header>
  <h1>Flight Operation Management System — Full Rebuild</h1>
</header>
<main class="container">
  <div class="grid">
    <section class="card">
      <h2>Search Flights</h2>
      <div class="search-row" style="margin-top:8px">
        <div style="flex:1;min-width:160px"><label>From</label><br><select id="fromSelect"></select></div>
        <div style="flex:1;min-width:160px"><label>To</label><br><select id="toSelect"></select></div>
        <div><label>Date</label><br><input id="dateInput" type="date"/></div>
        <div style="min-width:140px"><label>Passengers (1–10)</label><br><input id="paxInput" type="number" min="1" max="10" value="1" style="width:110px"/></div>
        <div style="align-self:flex-end"><button class="btn" id="searchBtn">Search</button></div>
      </div>

      <div style="margin-top:12px;display:flex;gap:8px;align-items:center">
        <label class="muted">Filter:</label>
        <select id="airlineFilter"><option value="">All Airlines</option></select>
        <label class="muted">Sort:</label>
        <select id="sortBy"><option value="time">Departure Time</option><option value="price">Price (low→high)</option></select>
      </div>

      <div id="results" class="results"></div>
    </section>

    <aside class="right-col">
      <div class="card">
        <h3>Passenger Details</h3>
        <form id="passengerForm">
          <label>Full Name</label><br>
          <input type="text" id="pName" required placeholder="e.g. Rahul Sharma"><br><br>
          <label>Age</label><br>
          <input type="number" id="pAge" required min="0" placeholder="e.g. 28"><br><br>
          <label>Email</label><br>
          <input type="email" id="pEmail" required placeholder="e.g. you@mail.com"><br><br>
          <label>Phone</label><br>
          <input type="text" id="pPhone" required placeholder="e.g. +91 98..."><br><br>
          <div style="display:flex;gap:8px">
            <button class="btn" type="submit">Save Passenger</button>
            <button class="ghost" type="button" id="addPassenger">Add Passenger</button>
            <button class="ghost" type="button" id="clearPassengers">Clear All</button>
          </div>
        </form>
        <div id="passengerList" style="margin-top:12px;font-size:14px;color:var(--muted)"></div>
      </div>

      <div class="card">
        <h3>Your Bookings</h3>
        <div id="bookingsList" class="bookings-list"></div>
        <div style="margin-top:10px;display:flex;gap:8px">
          <button class="ghost" id="exportBookings">Export JSON</button>
          <button class="ghost" id="clearBookings">Clear Bookings</button>
        </div>
      </div>
    </aside>
  </div>
</main>

<div id="modal" class="modal" role="dialog" aria-hidden="true">
  <div class="panel" id="modalPanel"></div>
</div>

<footer>Demo only — all data stored in browser localStorage. No external servers required.</footer>

<script>
// Full robust rebuild — generate many flights for every route so any from→to shows 10+ options
document.addEventListener('DOMContentLoaded', ()=>{
  const airports = {
    DEL: 'Delhi (DEL)', BOM: 'Mumbai (BOM)', BLR: 'Bengaluru (BLR)', MAA: 'Chennai (MAA)',
    HYD: 'Hyderabad (HYD)', CCU: 'Kolkata (CCU)', GOI: 'Goa (GOI)', PNQ: 'Pune (PNQ)',
    AMD: 'Ahmedabad (AMD)', JAI: 'Jaipur (JAI)', COK: 'Kochi (COK)', LKO: 'Lucknow (LKO)'
  };

  // helper to format times
  function addMinutes(time, mins){ const [h,m]=time.split(':').map(Number); const d=new Date(); d.setHours(h); d.setMinutes(m+mins); const hh=String(d.getHours()).padStart(2,'0'); const mm=String(d.getMinutes()).padStart(2,'0'); return `${hh}:${mm}`; }

  // airlines list
  const airlines = ['Air India','IndiGo','Vistara','SpiceJet','AirAsia India','GoFirst'];

  // generate flights programmatically so every distinct pair has many options
  const flights = [];
  const airportCodes = Object.keys(airports);
  airportCodes.forEach((from, idx)=>{
    airportCodes.forEach((to, jdx)=>{
      if(from===to) return;
      // generate 10 flights with different times/prices/airlines
      for(let k=0;k<10;k++){
        const baseHour = (6 + k*1) % 24; // spread across day
        const depart = `${String(baseHour).padStart(2,'0')}:${String((k*7)%60).padStart(2,'0')}`;
        // duration roughly 60-180 mins depending on index
        const duration = 60 + ((idx + jdx + k) % 180);
        const arrive = addMinutes(depart, duration);
        const airline = airlines[(idx + jdx + k) % airlines.length];
        const price = 2000 + ((idx + jdx + k) * 150) % 8000 + (Math.floor(Math.random()*500));
        const id = airline.split(' ')[0].slice(0,2).toUpperCase() + (100 + k + idx + jdx);
        flights.push({ id, airline, from, to, depart, arrive, price });
      }
    });
  });

  // --- Elements ---
  const fromSelect = document.getElementById('fromSelect');
  const toSelect = document.getElementById('toSelect');
  const dateInput = document.getElementById('dateInput');
  const paxInput = document.getElementById('paxInput');
  const searchBtn = document.getElementById('searchBtn');
  const resultsEl = document.getElementById('results');
  const airlineFilter = document.getElementById('airlineFilter');
  const sortBy = document.getElementById('sortBy');

  const passengerForm = document.getElementById('passengerForm');
  const pName = document.getElementById('pName');
  const pAge = document.getElementById('pAge');
  const pEmail = document.getElementById('pEmail');
  const pPhone = document.getElementById('pPhone');
  const addPassengerBtn = document.getElementById('addPassenger');
  const clearPassengersBtn = document.getElementById('clearPassengers');
  const passengerListEl = document.getElementById('passengerList');

  const bookingsListEl = document.getElementById('bookingsList');
  const exportBookingsBtn = document.getElementById('exportBookings');
  const clearBookingsBtn = document.getElementById('clearBookings');

  const modal = document.getElementById('modal');
  const modalPanel = document.getElementById('modalPanel');

  // populate selects
  Object.entries(airports).forEach(([code,name])=>{
    const o1 = document.createElement('option'); o1.value = code; o1.textContent = `${code} - ${name}`;
    const o2 = document.createElement('option'); o2.value = code; o2.textContent = `${code} - ${name}`;
    fromSelect.appendChild(o1); toSelect.appendChild(o2);
  });

  // populate airline filter
  new Set(flights.map(f=>f.airline)).forEach(a=>{ const o=document.createElement('option'); o.value=a; o.textContent=a; airlineFilter.appendChild(o); });

  dateInput.value = new Date().toISOString().slice(0,10);
  paxInput.value = 1;

  // seat counter stored
  function getSeatCounter(){ return parseInt(localStorage.getItem('foms.seatCounter')||'0',10); }
  function incrementSeatCounter(){ const v=getSeatCounter()+1; localStorage.setItem('foms.seatCounter',String(v)); return v; }
  function seatFromCounter(counter){ const letters=['A','B','C','D','E','F']; const row = Math.floor((counter-1)/6)+1; const col = letters[(counter-1)%6]; return row + col; }

  // passengers list
  let passengers = [];
  function clearForm(){ pName.value=''; pAge.value=''; pEmail.value=''; pPhone.value=''; }
  function refreshPassengerList(){ passengerListEl.innerHTML=''; if(passengers.length===0){ passengerListEl.textContent='No passengers added'; return; } passengers.forEach((p,i)=>{ const div=document.createElement('div'); div.className='passenger-entry'; div.innerHTML = `<strong>${i+1}. ${escapeHtml(p.name)}</strong> <span class="muted">(${escapeHtml(p.age)})</span> <button class="ghost" data-i="${i}">Delete</button>`; passengerListEl.appendChild(div); }); qa('.passenger-entry button').forEach(btn=>btn.addEventListener('click', e=>{ const i=parseInt(e.target.getAttribute('data-i'),10); passengers.splice(i,1); refreshPassengerList(); })); }

  function escapeHtml(s){ return String(s||'').replace(/[&<>'"]/g, c => ({'&':'&amp;','<':'&lt;','>':'&gt;',"'":'&#39;','"':'&quot;'}[c])); }
  function qa(sel, ctx=document){ return Array.from((ctx||document).querySelectorAll(sel)); }

  passengerForm.addEventListener('submit', e=>{ e.preventDefault(); const newP={ name:pName.value.trim(), age:pAge.value.trim(), email:pEmail.value.trim(), phone:pPhone.value.trim() }; if(!newP.name){ alert('Enter name'); return; } passengers=[newP]; refreshPassengerList(); clearForm(); });
  addPassengerBtn.addEventListener('click', ()=>{ const newP={ name:pName.value.trim(), age:pAge.value.trim(), email:pEmail.value.trim(), phone:pPhone.value.trim() }; if(!newP.name){ alert('Fill passenger details before adding'); return; } if(passengers.length>=50){ alert('Max passengers reached'); return; } passengers.push(newP); refreshPassengerList(); clearForm(); });
  clearPassengersBtn.addEventListener('click', ()=>{ if(confirm('Clear all entered passengers?')){ passengers=[]; refreshPassengerList(); } });

  paxInput.addEventListener('input', ()=>{ let v=parseInt(paxInput.value,10); if(isNaN(v)||v<1) v=1; if(v>50) v=50; paxInput.value=v; });

  // Render a flight item
  function renderFlightItem(f){ const el=document.createElement('div'); el.className='flight-item'; const left=document.createElement('div'); left.innerHTML=`<div style="font-weight:700">${escapeHtml(f.airline)} ${escapeHtml(f.id)}</div><div class='meta'>${escapeHtml(airports[f.from])} → ${escapeHtml(airports[f.to])}</div>`; const middle=document.createElement('div'); middle.innerHTML=`<div style="font-weight:600">${escapeHtml(f.depart)} — ${escapeHtml(f.arrive)}</div><div class='meta'>Price: ₹${f.price}</div>`; const right=document.createElement('div'); const bookBtn=document.createElement('button'); bookBtn.className='btn'; bookBtn.textContent='Book'; bookBtn.addEventListener('click', ()=> onBookFlight(f)); right.appendChild(bookBtn); el.appendChild(left); el.appendChild(middle); el.appendChild(right); return el; }

  // Search flights
  function searchFlights(){ resultsEl.innerHTML=''; const from=fromSelect.value; const to=toSelect.value; const date=dateInput.value; const pax=parseInt(paxInput.value,10)||1; const af=airlineFilter.value; const sort=sortBy.value; if(!from||!to){ resultsEl.textContent='Please select source and destination'; return; } if(from===to){ resultsEl.textContent='Source and destination must differ'; return; }
    let matches = flights.filter(f=>f.from===from && f.to===to);
    if(af) matches = matches.filter(m=>m.airline===af);
    if(sort==='time') matches.sort((a,b)=>a.depart.localeCompare(b.depart)); else matches.sort((a,b)=>a.price - b.price);
    if(matches.length===0){ resultsEl.textContent='No flights for this route'; return; }
    // show up to 20
    matches.slice(0,20).forEach(f=> resultsEl.appendChild(renderFlightItem(f)));
  }

  searchBtn.addEventListener('click', searchFlights);

  // Booking flow
  function onBookFlight(flight){ const requested = parseInt(paxInput.value,10)||1; if(passengers.length<requested){ openMissingPassengersModal(flight, requested-passengers.length); return; } const bookingPassengers = passengers.slice(0,requested).map(p=>({...p})); assignSeatsAndSaveBooking(flight, bookingPassengers); }

  function openMissingPassengersModal(flight, missingCount){ modalPanel.innerHTML=''; const h=document.createElement('h3'); h.textContent = `Add ${missingCount} passenger(s) for ${flight.airline} ${flight.id}`; modalPanel.appendChild(h); const form=document.createElement('form'); form.innerHTML='<div id="missingList"></div><div style="margin-top:12px;display:flex;gap:8px;justify-content:flex-end"><button type="button" id="cancelMissing" class="ghost">Cancel</button><button type="submit" class="btn">Save & Book</button></div>'; modalPanel.appendChild(form); const missingList=form.querySelector('#missingList'); const groups=[]; for(let i=0;i<missingCount;i++){ const g=document.createElement('div'); g.style.border='1px solid #eef2f7'; g.style.padding='10px'; g.style.borderRadius='8px'; g.style.marginBottom='8px'; g.innerHTML=`<div style="font-weight:600;margin-bottom:6px">Passenger ${passengers.length + i + 1}</div><div style="display:grid;grid-template-columns:1fr 80px;gap:8px;margin-bottom:8px"><input placeholder="Full name" class="m-name" required /><input placeholder="Age" class="m-age" type="number" min="0" required /></div><div style="display:grid;grid-template-columns:1fr 1fr;gap:8px"><input placeholder="Email" class="m-email" type="email" required /><input placeholder="Phone" class="m-phone" required /></div>`; missingList.appendChild(g); groups.push(g); }
    function close(){ modal.style.display='none'; modal.setAttribute('aria-hidden','true'); }
    form.querySelector('#cancelMissing').addEventListener('click', close);
    form.addEventListener('submit', e=>{ e.preventDefault(); for(const g of groups){ const name=g.querySelector('.m-name').value.trim(); const age=g.querySelector('.m-age').value.trim(); const email=g.querySelector('.m-email').value.trim(); const phone=g.querySelector('.m-phone').value.trim(); if(!name){ alert('Fill all passenger names'); return; } passengers.push({name,age,email,phone}); } refreshPassengerList(); close(); // proceed to booking with newly added passengers
      // book with updated passengers (use last booked flight)
      onBookFlight(flight);
    });
    modal.style.display='flex'; modal.setAttribute('aria-hidden','false'); }

  function assignSeatsAndSaveBooking(flight, bookingPassengers){ bookingPassengers.forEach(p=>{ const ctr=incrementSeatCounter(); p.seat=seatFromCounter(ctr); }); const booking={ id:'BK'+Date.now().toString(36).toUpperCase().slice(2), flightId:flight.airline+' '+flight.id, flightCode:flight.id, from:flight.from, to:flight.to, date:dateInput.value, price:flight.price, passengers:bookingPassengers, createdAt:new Date().toISOString() }; const arr=getBookings(); arr.unshift(booking); saveBookings(arr); renderBookings(); if(confirm(`Booked ${bookingPassengers.map(p=>p.name).join(', ')}. View booking?`)) openBookingViewer(booking.id); }

  // storage
  function getBookings(){ return JSON.parse(localStorage.getItem('foms.bookings')||'[]'); }
  function saveBookings(arr){ localStorage.setItem('foms.bookings', JSON.stringify(arr)); }

  function renderBookings(){ bookingsListEl.innerHTML=''; const arr=getBookings(); if(arr.length===0){ bookingsListEl.textContent='No bookings yet'; return; } arr.forEach(b=>{ const row=document.createElement('div'); row.style.borderBottom='1px solid #eef2f7'; row.style.padding='10px 0'; const left=document.createElement('div'); left.style.display='flex'; left.style.justifyContent='space-between'; left.innerHTML=`<div><strong>${escapeHtml(b.flightId)}</strong><div class='muted'>${escapeHtml(b.date)} · ₹${b.price}</div></div>`; const controls=document.createElement('div'); const viewBtn=document.createElement('button'); viewBtn.className='ghost'; viewBtn.textContent='View'; viewBtn.addEventListener('click', ()=> openBookingViewer(b.id)); const csvBtn=document.createElement('button'); csvBtn.className='ghost'; csvBtn.textContent='CSV'; csvBtn.addEventListener('click', ()=> downloadBookingCSV(b.id)); const printBtn=document.createElement('button'); printBtn.className='ghost'; printBtn.textContent='Print'; printBtn.addEventListener('click', ()=> printBooking(b.id)); const cancelBtn=document.createElement('button'); cancelBtn.className='ghost'; cancelBtn.textContent='Cancel'; cancelBtn.addEventListener('click', ()=>{ if(confirm('Cancel this booking?')) cancelBooking(b.id); }); controls.appendChild(viewBtn); controls.appendChild(csvBtn); controls.appendChild(printBtn); controls.appendChild(cancelBtn); row.appendChild(left); row.appendChild(controls); bookingsListEl.appendChild(row); }); }

  function cancelBooking(id){ const arr=getBookings().filter(b=>b.id!==id); saveBookings(arr); renderBookings(); }

  exportBookingsBtn.addEventListener('click', ()=>{ const data=JSON.stringify(getBookings(),null,2); const blob=new Blob([data],{type:'application/json'}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download='foms-bookings.json'; a.click(); URL.revokeObjectURL(url); });
  clearBookingsBtn.addEventListener('click', ()=>{ if(confirm('Clear all bookings?')){ localStorage.removeItem('foms.bookings'); renderBookings(); } });

  function openBookingViewer(id){ const b=getBookings().find(x=>x.id===id); if(!b) return alert('Booking not found'); modalPanel.innerHTML=''; const h=document.createElement('h3'); h.textContent=`Booking ${b.id}`; modalPanel.appendChild(h); const meta=document.createElement('div'); meta.innerHTML=`<div style="font-weight:600">${escapeHtml(b.flightId)}</div><div class='muted'>${escapeHtml(b.date)} · ₹${b.price} · Created: ${new Date(b.createdAt).toLocaleString()}</div>`; modalPanel.appendChild(meta); const table=document.createElement('table'); table.innerHTML='<thead><tr><th>#</th><th>Name</th><th>Age</th><th>Email</th><th>Phone</th><th>Seat</th></tr></thead>'; const tbody=document.createElement('tbody'); b.passengers.forEach((p,i)=>{ const tr=document.createElement('tr'); tr.innerHTML=`<td>${i+1}</td><td>${escapeHtml(p.name)}</td><td>${escapeHtml(p.age||'')}</td><td>${escapeHtml(p.email||'')}</td><td>${escapeHtml(p.phone||'')}</td><td>${escapeHtml(p.seat||'')}</td>`; tbody.appendChild(tr); }); table.appendChild(tbody); modalPanel.appendChild(table); const ctr=document.createElement('div'); ctr.style.marginTop='12px'; ctr.style.display='flex'; ctr.style.justifyContent='flex-end'; ctr.style.gap='8px'; const csvBtn=document.createElement('button'); csvBtn.className='btn'; csvBtn.textContent='Download CSV'; csvBtn.addEventListener('click', ()=> downloadBookingCSV(b.id)); const printBtn=document.createElement('button'); printBtn.className='ghost'; printBtn.textContent='Print / Save PDF'; printBtn.addEventListener('click', ()=> printBooking(b.id)); const closeBtn=document.createElement('button'); closeBtn.className='ghost'; closeBtn.textContent='Close'; closeBtn.addEventListener('click', closeModal); ctr.appendChild(csvBtn); ctr.appendChild(printBtn); ctr.appendChild(closeBtn); modalPanel.appendChild(ctr); modal.style.display='flex'; modal.setAttribute('aria-hidden','false'); function closeModal(){ modal.style.display='none'; modal.setAttribute('aria-hidden','true'); } }

  function downloadBookingCSV(id){ const b=getBookings().find(x=>x.id===id); if(!b) return alert('Booking not found'); const headers=['Name','Age','Email','Phone','Seat']; const rows=b.passengers.map(p=>[p.name||'',p.age||'',p.email||'',p.phone||'',p.seat||'']); const csv=[headers.join(','),...rows.map(r=>r.map(cell=>'"'+String(cell).replace(/"/g,'""')+'"').join(','))].join('\n'); const blob=new Blob([csv],{type:'text/csv'}); const url=URL.createObjectURL(blob); const a=document.createElement('a'); a.href=url; a.download=`${b.id}_passengers.csv`; a.click(); URL.revokeObjectURL(url); }

  function printBooking(id){ const b=getBookings().find(x=>x.id===id); if(!b) return alert('Booking not found'); const win=window.open('','_blank'); const html=`<!doctype html><html><head><title>Booking ${b.id}</title><style>body{font-family:Arial;padding:20px}table{border-collapse:collapse;width:100%}th,td{border:1px solid #ddd;padding:8px;text-align:left}</style></head><body><h2>Booking ${b.id}</h2><p><strong>${escapeHtml(b.flightId)}</strong><br>${b.date} · ₹${b.price}</p><table><thead><tr><th>#</th><th>Name</th><th>Age</th><th>Email</th><th>Phone</th><th>Seat</th></tr></thead><tbody>${b.passengers.map((p,i)=>`<tr><td>${i+1}</td><td>${escapeHtml(p.name)}</td><td>${escapeHtml(p.age||'')}</td><td>${escapeHtml(p.email||'')}</td><td>${escapeHtml(p.phone||'')}</td><td>${escapeHtml(p.seat||'')}</td></tr>`).join('')}</tbody></table></body></html>`; win.document.write(html); win.document.close(); win.focus(); }

  // initial render
  renderBookings(); refreshPassengerList();

  // expose for debugging
  window._foms = { flights, airports, getBookings, passengers };
});
</script>
</body>
</html>
