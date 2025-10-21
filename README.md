# attendance.html
<!doctype html>
<html lang="so">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Attendance System - Teachers Login</title>
<style>
  :root{--accent:#0b74de;--muted:#666}
  body{font-family:Inter,Segoe UI,Arial,Helvetica,sans-serif;margin:0;background:#f5f7fb;color:#111}
  .wrap{max-width:1000px;margin:28px auto;padding:18px;background:#fff;border-radius:10px;box-shadow:0 6px 20px rgba(18,20,30,0.06)}
  h1{margin:0 0 8px;font-size:22px}
  .muted{color:var(--muted);font-size:13px}
  .row{display:flex;gap:12px;align-items:center;flex-wrap:wrap;margin-bottom:14px}
  input,select,textarea{padding:8px 10px;border:1px solid #d6dbe6;border-radius:8px;font-size:14px}
  button{background:var(--accent);color:#fff;border:0;padding:8px 12px;border-radius:8px;cursor:pointer}
  button.ghost{background:transparent;color:var(--accent);border:1px solid var(--accent)}
  .login-box{max-width:520px;margin:40px auto;padding:18px;border-radius:10px;background:#fff;box-shadow:0 10px 40px rgba(2,8,23,0.06)}
  .muted-center{text-align:center;color:#666;margin-top:8px}
  table{width:100%;border-collapse:collapse;margin-top:12px}
  th,td{padding:8px;border-bottom:1px solid #eef0f4;font-size:14px;text-align:left}
  .idcol{width:100px}
  .actions{width:200px}
  .pill{display:inline-block;padding:4px 9px;border-radius:999px;font-weight:600;font-size:13px}
  .present{background:#e6fbec;color:#0a8a3e}
  .absent{background:#fff2f2;color:#e03a3a;border:1px solid #ffd8d8}
  .note{font-size:13px;color:#444;margin-top:8px}
  .right{margin-left:auto}
  .danger{background:#ff4d4f}
  @media(max-width:800px){.row{flex-direction:column;align-items:stretch}}
</style>
</head>
<body>
<div id="app"></div>
<script>
/*
  index.html - Attendance single-file app
  - Teacher register / login by (teacherNumber + PIN)
  - Each teacher has classes (Class 1..12) and can add students per class (max 90)
  - Attendance saved per teacher: key = teacherNumber|date|subject|class
  - Storage: localStorage under prefix 'att_v3_'
*/

const PREFIX = 'att_v3_';
const DB = {
  get(k, fallback=null){ const v = localStorage.getItem(PREFIX + k); return v ? JSON.parse(v) : fallback; },
  set(k,v){ localStorage.setItem(PREFIX + k, JSON.stringify(v)); },
  remove(k){ localStorage.removeItem(PREFIX + k); }
};

let currentTeacher = DB.get('currentTeacher', null); // teacherNumber string
let teachers = DB.get('teachers', {}); // { teacherNumber: { pin, name, school, classes: { "Class 1": [{id,name},...] } } }
let attendance = DB.get('attendance', {}); // { "teacher|date|subject|class": [{id,name,status}] }

function ensureTeacherClasses(obj){
  if(!obj.classes) obj.classes = {};
  for(let i=1;i<=12;i++){
    const key = 'Class ' + i;
    if(!Array.isArray(obj.classes[key])) obj.classes[key] = [];
  }
}

function saveAll(){
  DB.set('teachers', teachers);
  DB.set('attendance', attendance);
  if(currentTeacher) DB.set('currentTeacher', currentTeacher); else DB.remove('currentTeacher');
}

/* DOM helpers */
function el(t,a={},...ch){ const e = document.createElement(t); for(const k in a){ if(k.startsWith('on')) e.addEventListener(k.slice(2), a[k]); else if(k==='html') e.innerHTML=a[k]; else e.setAttribute(k,a[k]); } ch.forEach(c=>{ if(c==null) return; e.append(typeof c==='string' ? document.createTextNode(c) : c); }); return e; }

function showLogin(){
  const root = document.getElementById('app'); root.innerHTML = '';
  const box = el('div',{class:'login-box'},
    el('div',{class:'center'}, el('h1',{}, 'Attendance System — Login / Register')),
    el('div',{style:'margin-top:10px'},
       el('label',{}, 'Teacher Number'),
       el('input',{type:'text', id:'login_number', placeholder:'Tusaale: 1001', style:'width:100%;margin-top:6px'})
    ),
    el('div',{style:'margin-top:8px'},
       el('label',{}, 'PIN (4-6 digits)'),
       el('input',{type:'password', id:'login_pin', placeholder:'PIN', style:'width:100%;margin-top:6px'})
    ),
    el('div',{style:'margin-top:12px', class:'row'},
       el('button',{onclick: doLogin}, 'Login'),
       el('button',{class:'ghost', onclick: showRegister}, 'Register New Teacher')
    ),
    el('div',{class:'muted-center'}, 'Haddii aad hore u diiwaangashan tahay, gala. Haddii kale dooro Register.')
  );
  root.appendChild(box);
}

function showRegister(){
  const root = document.getElementById('app'); root.innerHTML = '';
  const box = el('div',{class:'login-box'},
    el('div',{class:'center'}, el('h1',{}, 'Register Teacher')),
    el('div',{style:'margin-top:10px'},
       el('label',{}, 'School Name'),
       el('input',{type:'text', id:'reg_school', placeholder:'Magaca School', style:'width:100%;margin-top:6px'})
    ),
    el('div',{style:'margin-top:8px'},
       el('label',{}, 'Teacher Name'),
       el('input',{type:'text', id:'reg_name', placeholder:'Magaca Macalinka', style:'width:100%;margin-top:6px'})
    ),
    el('div',{class:'row', style:'margin-top:8px'},
       el('div',{}, el('label',{}, 'Teacher Number'), el('input',{type:'text', id:'reg_number', placeholder:'1001'})),
       el('div',{}, el('label',{}, 'PIN'), el('input',{type:'password', id:'reg_pin', placeholder:'4-6 digits'}))
    ),
    el('div',{style:'margin-top:12px', class:'row'},
       el('button',{onclick: doRegister}, 'Register'),
       el('button',{class:'ghost', onclick: showLogin}, 'Back to Login')
    )
  );
  root.appendChild(box);
}

function doRegister(){
  const school = document.getElementById('reg_school').value.trim();
  const name = document.getElementById('reg_name').value.trim();
  const num = document.getElementById('reg_number').value.trim();
  const pin = document.getElementById('reg_pin').value.trim();
  if(!school||!name||!num||!pin) return alert('Fadlan buuxi dhammaan goobaha.');
  if(teachers[num]) return alert('Teacher number hore ayaa diiwaan ah.');
  if(pin.length < 4 || pin.length > 8) return alert('PIN waa in uu u dhaxeeyaa 4-8 characters.');
  teachers[num] = { school, name, pin, classes: {} };
  ensureTeacherClasses(teachers[num]);
  saveAll();
  alert('Registration completed. Fadlan login hadda.');
  showLogin();
}

function doLogin(){
  const num = document.getElementById('login_number').value.trim();
  const pin = document.getElementById('login_pin').value.trim();
  if(!num||!pin) return alert('Geli teacher number iyo PIN.');
  const t = teachers[num];
  if(!t) return alert('Teacher number ma jiro. Fadlan iska diiwaangeli marka hore.');
  if(t.pin !== pin) return alert('PIN khaldan.');
  currentTeacher = num;
  DB.set('currentTeacher', currentTeacher);
  renderMain();
}

/* Main UI */
function renderMain(){
  if(!currentTeacher){ showLogin(); return; }
  const teacher = teachers[currentTeacher];
  ensureTeacherClasses(teacher); // just in case
  const root = document.getElementById('app'); root.innerHTML = '';
  const header = el('div',{class:'row'},
    el('div',{},
       el('h1',{}, `Attendance — ${teacher.school}`),
       el('div',{class:'muted'}, `Macalin: ${teacher.name} • Number: ${currentTeacher}`)
    ),
    el('div',{}, el('button',{class:'ghost', onclick: logout}, 'Logout'))
  );

  // controls: date, subject, class select
  const classSelect = el('select',{id:'sel_class', onchange: renderClass});
  for(let i=1;i<=12;i++){
    const nm = 'Class ' + i;
    classSelect.appendChild(el('option',{value:nm}, nm));
  }
  const controls = el('div',{class:'row'},
    el('div',{}, el('label',{}, 'Taariikh'), el('input',{type:'date', id:'att_date', value: new Date().toISOString().slice(0,10)})),
    el('div',{}, el('label',{}, 'Mawduuc'), el('input',{type:'text', id:'att_subject', placeholder:'Tusaale: Xisaab'})),
    el('div',{}, el('label',{}, 'Fasalka'), classSelect),
    el('div',{}, el('label',{}, 'Section (optional)'), el('input',{type:'text', id:'att_section', placeholder:'3A'}))
  );

  // add student to current class
  const addStudentForm = el('div',{},
    el('h3',{}, 'Ku dar Arday (fasalka hadda dooran)'),
    el('div',{class:'row'},
      el('input',{type:'text', id:'new_student_id', placeholder:'ID (Tusaale: 1)'}),
      el('input',{type:'text', id:'new_student_name', placeholder:'Magaca Ardayga'}),
      el('button',{onclick: addStudent}, 'Ku dar')
    ),
    el('div',{class:'note'}, 'Fasal walba ilaa 90 arday.')
  );

  // toolbar
  const toolbar = el('div',{class:'row'},
    el('button',{onclick: saveAttendance}, 'Save Attendance'),
    el('button',{onclick: exportCSV}, 'Export CSV'),
    el('button',{class:'danger', onclick: clearClass}, 'Tirtir Fasalka'),
    el('div',{style:'flex:1'}), // spacer
    el('button',{onclick: showRecords}, 'Tus Record-yadii Hore')
  );

  root.appendChild(el('div',{class:'wrap'}, header, controls, addStudentForm, toolbar, el('div',{id:'class_container'})));
  // set selected class to first
  document.getElementById('sel_class').value = 'Class 1';
  renderClass();
}

/* actions */
function logout(){ if(confirm('Ka bax?')){ currentTeacher = null; DB.remove('currentTeacher'); renderMain(); } }

function getCurrentClass(){ return document.getElementById('sel_class').value; }

function addStudent(){
  const id = document.getElementById('new_student_id').value.trim();
  const name = document.getElementById('new_student_name').value.trim();
  if(!id||!name) return alert('Geli ID iyo Magaca.');
  const teacher = teachers[currentTeacher];
  const cls = getCurrentClass();
  const list = teacher.classes[cls] || [];
  if(list.some(s=>s.id===id)) return alert('ID hore ayaa jira fasalkan.');
  if(list.length >= 90) return alert('Fasalkani wuxuu gaadhay xadka 90 arday.');
  list.push({id,name});
  teacher.classes[cls] = list;
  teachers[currentTeacher] = teacher;
  saveAll();
  document.getElementById('new_student_id').value=''; document.getElementById('new_student_name').value='';
  renderClass();
}

function deleteStudent(id){
  if(!confirm('Ma hubtaa inaad tirtiri rabto ardaygan?')) return;
  const teacher = teachers[currentTeacher];
  const cls = getCurrentClass();
  teacher.classes[cls] = teacher.classes[cls].filter(s=>s.id!==id);
  teachers[currentTeacher]=teacher; saveAll(); renderClass();
}

function renderClass(){
  const container = document.getElementById('class_container');
  container.innerHTML = '';
  const teacher = teachers[currentTeacher];
  const cls = getCurrentClass();
  const students = teacher.classes[cls] || [];
  if(students.length === 0){
    container.appendChild(el('div',{class:'muted'}, `Ma jiraan arday ku jira ${cls}. Ku dar arday si aad u bilowdo.`));
    return;
  }

  const date = document.getElementById('att_date').value || new Date().toISOString().slice(0,10);
  const subject = (document.getElementById('att_subject')||{}).value || '—';
  const key = `${currentTeacher}|${date}|${subject}|${cls}`;
  const existing = attendance[key] || students.map(s=>({id:s.id,name:s.name,status:'absent'}));
  const mapEx = {}; existing.forEach(r=>mapEx[r.id]=r);

  const rows = students.map(s => mapEx[s.id] ? {...mapEx[s.id], name:s.name} : {id:s.id,name:s.name,status:'absent'});

  const tbl = el('table',{},
    el('thead',{}, el('tr',{}, el('th',{class:'idcol'}, 'ID'), el('th',{}, 'Magaca'), el('th',{}, 'Xaalad'), el('th',{}, 'Action'))),
    el('tbody',{}, ...rows.map(r=>{
      const pill = el('span',{class:'pill ' + (r.status==='present' ? 'present':'absent')}, r.status==='present' ? 'Joog' : 'Maqan');
      const checkbox = el('input',{type:'checkbox', checked: r.status==='present', onchange: (e)=> toggleStatus(r.id, e.target.checked)});
      return el('tr',{}, el('td',{}, r.id), el('td',{}, r.name), el('td',{}, pill, ' ', checkbox), el('td',{class:'actions'},
        el('button',{onclick:()=>forcePresent(r.id)}, 'Joog'),
        ' ',
        el('button',{onclick:()=>forceAbsent(r.id)}, 'Maqan'),
        ' ',
        el('button',{onclick:()=>deleteStudent(r.id), class:'ghost'}, 'Tirtir')
      ));
    })))
  ;

  container.appendChild(el('div',{}, el('div',{class:'muted'}, `Taariikh: ${date} • Mawduuc: ${subject} • ${cls} (Arday: ${students.length})`)));
  container.appendChild(tbl);
}

function toggleStatus(id, isPresent){
  const date = document.getElementById('att_date').value || new Date().toISOString().slice(0,10);
  const subject = (document.getElementById('att_subject')||{}).value || '—';
  const cls = getCurrentClass();
  const key = `${currentTeacher}|${date}|${subject}|${cls}`;
  const students = teachers[currentTeacher].classes[cls] || [];
  const rec = attendance[key] || students.map(s=>({id:s.id,name:s.name,status:'absent'}));
  const idx = rec.findIndex(r=>r.id===id);
  if(idx>=0) rec[idx].status = isPresent ? 'present' : 'absent';
  else rec.push({id, name: students.find(s=>s.id===id).name, status: isPresent ? 'present' : 'absent'});
  attendance[key] = rec; saveAll(); renderClass();
}

function forcePresent(id){ toggleStatus(id,true); }
function forceAbsent(id){ toggleStatus(id,false); }

function saveAttendance(){
  const date = document.getElementById('att_date').value || new Date().toISOString().slice(0,10);
  const subject = (document.getElementById('att_subject')||{}).value.trim();
  if(!subject) return alert('Geli mawduuc marka hore.');
  const cls = getCurrentClass();
  const key = `${currentTeacher}|${date}|${subject}|${cls}`;
  const students = teachers[currentTeacher].classes[cls] || [];
  const current = students.map(s=>{
    const existing = (attendance[key]||[]).find(r=>r.id===s.id);
    return { id:s.id, name:s.name, status: existing ? existing.status : 'absent' };
  });
  attendance[key] = current; saveAll(); alert('Attendance la kaydiyey.');
}

function exportCSV(){
  const date = document.getElementById('att_date').value || new Date().toISOString().slice(0,10);
  const subject = (document.getElementById('att_subject')||{}).value || 'subject';
  const cls = getCurrentClass();
  const key = `${currentTeacher}|${date}|${subject}|${cls}`;
  const rec = attendance[key] || (teachers[currentTeacher].classes[cls]||[]).map(s=>({id:s.id,name:s.name,status:'absent'}));
  const header = ['School','TeacherNumber','TeacherName','Date','Subject','Class','StudentID','StudentName','Status'];
  const rows = rec.map(r=> [ teachers[currentTeacher].school, currentTeacher, teachers[currentTeacher].name, date, subject, cls, r.id, r.name, r.status ]);
  let csv = header.join(',') + '\n' + rows.map(r => r.map(c => `"${String(c).replace(/"/g,'""')}"`).join(',')).join('\n');
  const blob = new Blob([csv], {type:'text/csv;charset=utf-8;'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a'); a.href = url;
  a.download = `attendance_${currentTeacher}_${cls}_${date}_${subject.replace(/\s+/g,'_')}.csv`;
  a.click(); URL.revokeObjectURL(url);
}

function clearClass(){
  if(!confirm('Tani waxay tirtiri doontaa dhammaan ardayda fasalka hadda dooran. Ma hubtaa?')) return;
  const cls = getCurrentClass();
  teachers[currentTeacher].classes[cls] = [];
  // remove attendance records for this teacher+class
  Object.keys(attendance).forEach(k=>{ if(k.startsWith(currentTeacher + '|') && k.endsWith('|' + cls)) delete attendance[k]; });
  saveAll(); renderClass();
}

function showRecords(){
  const keys = Object.keys(attendance).filter(k => k.startsWith(currentTeacher + '|'));
  if(keys.length===0) return alert('Ma jiraan record hore macalinkan.');
  const lines = keys.map(k=>{
    const parts = k.split('|'); // teacher|date|subject|class
    const date = parts[1]||'';
    const subject = parts[2]||'';
    const cls = parts[3]||'';
    const present = (attendance[k]||[]).filter(r=>r.status==='present').length;
    const total = (attendance[k]||[]).length;
    return `${date} • ${subject} • ${cls} — Joog: ${present}/${total}`;
  }).join('\n');
  alert('Record-yada:\n\n' + lines);
}

/* init */
(function init(){
  teachers = DB.get('teachers', {});
  attendance = DB.get('attendance', {});
  currentTeacher = DB.get('currentTeacher', null);
  // ensure existing teachers have class buckets
  for(const k in teachers) ensureTeacherClasses(teachers[k]);
  if(currentTeacher) renderMain(); else showLogin();
})();
</script>
</body>
</html>
