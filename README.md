<!-- Comentarios emcambezados -->
# Mi titulo h1
## Mi titulo h2
### Mi titulo h3
#### Mi titulo h4
##### Mi titulo h5
###### Mi titulo h6

<!-- Modificacion del texto -->
Este texto escrito es *italica*

Este texto con lineas mas gruesas **negrito**

este es un texto ~~tachado~~

<!-- UL -->
* Manzana
    * Manzana verde
    * Manzana roja
* Naranja
* Platano

1. manzana
    1. Manzana Roja
    2. Manzana verde

2. naranja
3. platano

[Google.com](https://www.google.com)

[Google.com](https://www.google.com, "Sapo Url")

> Esta es un cita

---
___

`console.log('hello world')`

```javascript
// =============================================================================
//  cprj_project_phase_import(pProj_id)
// =============================================================================
//
//  Used in form button of cprj_project object.
//
//  Imports Kanban phases to project phases.
//  If a phase already has a related Kanban phase id, the information from the
//  phase is updated to new possible values.
//
//  If the Kanban phase trying to be created has a code already existing in
//  project phases, a constraint exception will be raised.
//
// =============================================================================
function cprj_project_phase_import(pProj_id) {
    
    let mRsKbProjectChanges = Ax.db.call('cprj_get_kb_project_id');
    
    // Obtain Kanban board id defined in cprj_project
    let mObjProject = Ax.db.executeQuery(`
        <select>
            <columns>
                cprj_project.proj_code,
                cprj_project.proj_name,
                cprj_project.proj_empcode,
                cprj_project.proj_tercer,
                cprj_project.kb_board_id
            </columns>
            <from table='cprj_project'/>
            <where>
                cprj_project.proj_id = ?
            </where>
        </select>`, pProj_id).toOne();

    // If no Kanban board id was defined, raise exception
    if (mObjProject.kb_board_id == null) {
        mRsKbProjectChanges.rows().add(
            {
                action: 'BOARD_ID_NOT_FOUND', 
                proj_code: mObjProject.proj_code, 
                proj_name: mObjProject.proj_name, 
                proj_empcode: mObjProject.proj_empcode,
                proj_tercer: mObjProject.proj_tercer,
                phase_code: null,
                comment: `No Kanban board id was found in project [${pProj_id}].`
            });
        return mRsKbProjectChanges;
    }

    // =============================================================================
    // Kanban connection to obtain Kanban phases
    // =============================================================================
    let dbKanban = Ax.db.of('kanban_deister');
    let rsPhases = dbKanban.executeQuery(`
        <select>
            <columns>
                kb_board_phase.board_id,
                kb_board_phase.phase_id,
                kb_board_phase.phase_code,
                kb_board_phase.phase_name,
                kb_board_phase.phase_desc,
                kb_board_phase.phase_status,
                kb_board_phase.phase_exp_start_date,
                kb_board_phase.phase_exp_end_date,
                kb_board_phase.phase_eff_start_date,
                CASE WHEN kb_board_phase.phase_status = 1 THEN NULL
                    ELSE kb_board_phase.phase_eff_end_date
                END phase_eff_end_date,
                kb_board_phase.phase_exp_hours,
                kb_board_phase.phase_set_coverage
            </columns>
            <from table='kb_board_phase'>
                <join table='kb_board'>
                    <on>kb_board_phase.board_id = kb_board.board_id</on>
                </join>
            </from>
            <where>
                kb_board_phase.board_id = ?
            </where>
        </select>`, mObjProject.kb_board_id);
    
    rsPhases.forEach(mPhase => {
        // =============================================================================
        // Project phase object data
        // =============================================================================
        let cprj_project_phase = {
            proj_id: pProj_id,
            phase_code: mPhase.phase_code,
            phase_name: mPhase.phase_name,
            phase_desc: mPhase.phase_desc,
            phase_status: mPhase.phase_status,
            phase_exp_datestart: mPhase.phase_exp_start_date,
            phase_exp_dateend: mPhase.phase_exp_end_date,
            phase_datestart: mPhase.phase_eff_start_date,
            phase_dateend: mPhase.phase_eff_end_date,
            phase_exp_hours: mPhase.phase_exp_hours,
            phase_exp_coverage: mPhase.phase_set_coverage,
            kb_phase_id: mPhase.phase_id,
            user_updated: Ax.ext.user.getCode(),
            date_updated: new Ax.sql.Date()
        };

        // If a Kanban phase id is found in more than one project phase, raise expection
        let mPhase_id = null;
        try {
            mPhase_id = Ax.db.executeGet(`
                <select>
                    <columns>
                        cprj_project_phase.phase_id
                    </columns>
                    <from table='cprj_project_phase'/>
                    <where>
                        cprj_project_phase.kb_phase_id = ?
                    </where>
                </select>`, mPhase.phase_id);
        } catch (e) {
            throw new Ax.ext.Exception('KB_PHASE_REPEATED', 'Kanban phase found in more than one project phase.');
        }
        
        // If Kanban phase is not found in project phases, new phase is inserted,
        // if found, existing phase is updated
        if (mPhase_id == null) {
            cprj_project_phase.phase_id = 0;
            cprj_project_phase.user_created = Ax.ext.user.getCode();
            cprj_project_phase.date_created = new Ax.sql.Date();
            
            mPhase_id = Ax.db.executeGet(`
                <select>
                    <columns>
                        cprj_project_phase.phase_id
                    </columns>
                    <from table='cprj_project_phase'/>
                    <where>
                        cprj_project_phase.phase_code = ?
                        AND cprj_project_phase.proj_id = ?
                        AND cprj_project_phase.kb_phase_id IN (NULL, 0)
                    </where>
                </select>`, mPhase.phase_code, pProj_id);
                
            if (mPhase_id == null) {
                // =============================================================================
                // Insert new phase
                // =============================================================================
                Ax.db.insert('cprj_project_phase', cprj_project_phase);
                
                mRsKbProjectChanges.rows().add(
                {
                    action: 'CREADA', 
                    proj_code: mObjProject.proj_code, 
                    proj_name: mObjProject.proj_name, 
                    proj_empcode: mObjProject.proj_empcode,
                    proj_tercer: mObjProject.proj_tercer,
                    phase_code: cprj_project_phase.phase_code,
                    comment: `OK.`
                });
            } else {
                cprj_project_phase.phase_id = mPhase_id;
                
                // =============================================================================
                // Update existing phase
                // =============================================================================
                Ax.db.update('cprj_project_phase', cprj_project_phase);
                
                mRsKbProjectChanges.rows().add(
                {
                    action: 'CANVI', 
                    proj_code: mObjProject.proj_code, 
                    proj_name: mObjProject.proj_name, 
                    proj_empcode: mObjProject.proj_empcode,
                    proj_tercer: mObjProject.proj_tercer,
                    phase_code: cprj_project_phase.phase_code,
                    comment: `S'ha actualitzat la fase.`
                });
            }
        } else {
            cprj_project_phase.phase_id = mPhase_id;
            
            // =============================================================================
            // Update existing phase
            // =============================================================================
            Ax.db.update('cprj_project_phase', cprj_project_phase);
            
            mRsKbProjectChanges.rows().add(
                {
                    action: 'CANVI', 
                    proj_code: mObjProject.proj_code, 
                    proj_name: mObjProject.proj_name, 
                    proj_empcode: mObjProject.proj_empcode,
                    proj_tercer: mObjProject.proj_tercer,
                    phase_code: cprj_project_phase.phase_code,
                    comment: `S'ha actualitzat la fase.`
                });
        }
    });
    
    return mRsKbProjectChanges;
}
```

```python
print("hello world")
```

```html
<h1> hello world </h1>
```

| Tables | Are | Cool |
|--------|-----|------|
| col 1  |right| 7    |
| col 2  |green| 8    |


## Genos y Tornado
![anime](cc57d798-b694-4018-8a88-797c2965d9c4.jfif "genos y tornado")


<!-- GITHUB MARKDOWN -->
* [x] Task 1
* [] Task 2
* [] Task 3
