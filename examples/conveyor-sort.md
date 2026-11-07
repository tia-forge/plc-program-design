# 附：分拣工位（SCL 源 + 仿真用例）

2026-11-07 实测：TiaLink 1.5.1.0 + 博途 V19 + PLCSIM Advanced 6.0，编译 0 错 0 警、下载 0 错，
用例 8 条 + 停机用例 3 条全 PASS。下面是完整源码与用例脚本。

点位（与本技能第 1 步的模板一致）：DI `%I0.0–%I1.2`、DQ `%Q0.0–%Q0.6`。

| 符号 | 地址 | 说明 |
|---|---|---|
| DI_Conv1_PartAtEntry / DI_Conv1_PartAtSort | %I0.0 / %I0.1 | 入口光电 / 分拣位光电 |
| DI_Sys_SafetyOK | %I0.2 | 安全回路正常（安全继电器辅助触点） |
| DI_Hmi_Start / Stop / Reset | %I0.3 / .4 / .5 | 面板按钮 |
| DI_CylA_ExtEnd / RetEnd、DI_CylB_ExtEnd / RetEnd | %I0.6–%I1.1 | 气缸到位 |
| DI_Qual_ResultOK | %I1.2 | 质量结果 1=合格 |
| DO_Conv1_Run | %Q0.0 | 输送机运行 |
| DO_CylA_Extend / Retract、DO_CylB_Extend / Retract | %Q0.1–%Q0.4 | 气缸 |
| DO_Sys_FaultLamp / ReadyLamp | %Q0.5 / %Q0.6 | 指示灯 |

## 1. SCL 外部源（一次导入 3 个块 + 1 个实例 DB）

```pascal
FUNCTION_BLOCK "FB_Cylinder"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_xExtend : Bool;
      i_xRetract : Bool;
      i_xReset : Bool;
      i_xPermit : Bool;          // 许可：FALSE 立即回 Idle、清输出、不报故障
      i_xExtLimit : Bool;
      i_xRetLimit : Bool;
      i_tTimeOut : Time := T#3S;
   END_VAR
   VAR_OUTPUT
      q_xExtendCmd : Bool;
      q_xRetractCmd : Bool;
      q_xBusy : Bool;
      q_xFault : Bool;
      q_wFaultCode : Word;
   END_VAR
   VAR
      stat_wState : Int;
      stat_tTO : TON_TIME;
      stat_xResetOld : Bool;
   END_VAR
BEGIN
     // 块首统一默认值：任何路径都不会残留上一个扫描周期的输出
     #q_xExtendCmd := FALSE;
     #q_xRetractCmd := FALSE;
     #q_xBusy := FALSE;
     #q_xFault := FALSE;

     IF NOT #i_xPermit THEN          // 撤许可 = 撤命令：立即回 Idle，超时不算故障
         #stat_wState := 0;
         #stat_tTO(IN := FALSE, PT := #i_tTimeOut);
         #q_wFaultCode := 0;
     END_IF;

     CASE #stat_wState OF
         0: // Idle
             #stat_tTO(IN := FALSE, PT := #i_tTimeOut);
             #q_wFaultCode := 0;
             IF #i_xExtend AND NOT #i_xRetract THEN
                 #stat_wState := 10;
             ELSIF #i_xRetract AND NOT #i_xExtend THEN
                 #stat_wState := 20;
             END_IF;

         10: // 伸出
             #q_xBusy := TRUE;
             #q_xExtendCmd := TRUE;
             #stat_tTO(IN := TRUE, PT := #i_tTimeOut);
             IF #i_xExtLimit THEN
                 #stat_wState := 0;
             ELSIF #stat_tTO.Q THEN
                 #q_wFaultCode := 16#0201;
                 #stat_wState := 90;
             END_IF;

         20: // 缩回
             #q_xBusy := TRUE;
             #q_xRetractCmd := TRUE;
             #stat_tTO(IN := TRUE, PT := #i_tTimeOut);
             IF #i_xRetLimit THEN
                 #stat_wState := 0;
             ELSIF #stat_tTO.Q THEN
                 #q_wFaultCode := 16#0202;
                 #stat_wState := 90;
             END_IF;

         90: // Fault：只保留复位出路
             #q_xFault := TRUE;
             #stat_tTO(IN := FALSE, PT := #i_tTimeOut);
             IF #i_xReset AND NOT #stat_xResetOld THEN
                 #q_wFaultCode := 0;
                 #stat_wState := 0;
             END_IF;
     END_CASE;

     #stat_xResetOld := #i_xReset;
END_FUNCTION_BLOCK

FUNCTION_BLOCK "FB_Conveyor"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_xRun : Bool;
      i_xFeedback : Bool;
      i_xReset : Bool;
      i_tFbTimeOut : Time := T#2S;
   END_VAR
   VAR_OUTPUT
      q_xMotor : Bool;
      q_xFault : Bool;
      q_wFaultCode : Word;
   END_VAR
   VAR
      stat_xRunOld : Bool;
      stat_tTO : TON_TIME;
   END_VAR
BEGIN
     #q_xMotor := FALSE;
     #q_xFault := FALSE;

     IF #stat_xRunOld AND NOT #i_xRun THEN
         #q_wFaultCode := 0;          // 停机即清反馈监控
     END_IF;

     IF #i_xRun THEN
         #q_xMotor := TRUE;
         #stat_tTO(IN := TRUE, PT := #i_tFbTimeOut);
         IF NOT #i_xFeedback THEN
             IF #stat_tTO.Q THEN
                 #q_wFaultCode := 16#0301;   // 运行无反馈
                 #q_xFault := TRUE;
                 #q_xMotor := FALSE;
             END_IF;
         ELSE
             #stat_tTO(IN := FALSE, PT := #i_tFbTimeOut);
         END_IF;
     ELSE
         #stat_tTO(IN := FALSE, PT := #i_tFbTimeOut);
     END_IF;

     IF #q_xFault THEN
         #q_xMotor := FALSE;
         IF #i_xReset THEN
             #q_wFaultCode := 0;
             #stat_tTO(IN := FALSE, PT := #i_tFbTimeOut);
         END_IF;
     END_IF;

     #stat_xRunOld := #i_xRun;
END_FUNCTION_BLOCK

FUNCTION_BLOCK "FB_SortStation"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   VAR_INPUT
      i_xStart : Bool;
      i_xStop : Bool;
      i_xReset : Bool;
      i_xSafetyOK : Bool;
      i_xPartAtEntry : Bool;
      i_xPartAtSort : Bool;
      i_xQualityOK : Bool;
      i_xCylA_ExtEnd : Bool;
      i_xCylA_RetEnd : Bool;
      i_xCylB_ExtEnd : Bool;
      i_xCylB_RetEnd : Bool;
      i_tFeedTimeOut : Time := T#5S;
      i_tHoldTime : Time := T#800MS;
   END_VAR
   VAR_OUTPUT
      q_xConveyor : Bool;
      q_xCylA_Extend : Bool;
      q_xCylB_Extend : Bool;
      q_xReady : Bool;
      q_xFault : Bool;
      q_wState : Word;
      q_wFaultCode : Word;
   END_VAR
   VAR
      stat_wState : Int;
      stat_tFeed : TON_TIME;
      stat_tHold : TON_TIME;
      stat_xQualityNG : Bool;
      stat_xResetOld : Bool;
      stat_aConv1 : "FB_Conveyor";
      stat_aCylA : "FB_Cylinder";
      stat_aCylB : "FB_Cylinder";
   END_VAR
BEGIN
     // 块首默认值
     #q_xConveyor := FALSE;
     #q_xCylA_Extend := FALSE;
     #q_xCylB_Extend := FALSE;
     #q_xReady := FALSE;
     #q_xFault := FALSE;

     // —— 设备层实例：只给命令与反馈，超时由设备层自己判 ——
     #stat_aConv1(i_xRun := (#stat_wState = 10),
                  i_xFeedback := #i_xPartAtEntry OR #i_xPartAtSort,
                  i_xReset := #i_xReset,
                  i_tFbTimeOut := T#2S);

     #stat_aCylA(i_xExtend := (#stat_wState = 20 AND #stat_xQualityNG = FALSE),
                 i_xRetract := (#stat_wState = 30),
                 i_xReset := #i_xReset,
                 i_xPermit := #i_xSafetyOK AND NOT #i_xStop,
                 i_xExtLimit := #i_xCylA_ExtEnd,
                 i_xRetLimit := #i_xCylA_RetEnd,
                 i_tTimeOut := T#3S);

     #stat_aCylB(i_xExtend := (#stat_wState = 20 AND #stat_xQualityNG = TRUE),
                 i_xRetract := (#stat_wState = 30),
                 i_xReset := #i_xReset,
                 i_xPermit := #i_xSafetyOK AND NOT #i_xStop,
                 i_xExtLimit := #i_xCylB_ExtEnd,
                 i_xRetLimit := #i_xCylB_RetEnd,
                 i_tTimeOut := T#3S);

     // 设备层故障上抛
     IF #stat_aConv1.q_xFault OR #stat_aCylA.q_xFault OR #stat_aCylB.q_xFault THEN
         IF #stat_wState < 90 THEN
             #q_wFaultCode := #stat_aConv1.q_wFaultCode
                           OR #stat_aCylA.q_wFaultCode
                           OR #stat_aCylB.q_wFaultCode;
             #stat_wState := 90;
         END_IF;
     END_IF;

     CASE #stat_wState OF
         0: // Idle
             #stat_tFeed(IN := FALSE, PT := #i_tFeedTimeOut);
             #stat_tHold(IN := FALSE, PT := #i_tHoldTime);
             #q_wFaultCode := 0;
             #q_xReady := #i_xSafetyOK;
             #stat_xQualityNG := FALSE;
             IF #i_xStart AND #i_xSafetyOK THEN
                 #stat_wState := 10;
             END_IF;

         10: // 进料
             IF #i_xPartAtEntry THEN
                 #stat_xQualityNG := NOT #i_xQualityOK;   // 入口锁存质量结果
             END_IF;
             #stat_tFeed(IN := TRUE, PT := #i_tFeedTimeOut);
             IF #i_xPartAtSort AND #i_xPartAtEntry = FALSE THEN
                 #stat_tFeed(IN := FALSE, PT := #i_tFeedTimeOut);
                 #stat_wState := 20;
             ELSIF #stat_tFeed.Q THEN
                 #q_wFaultCode := 16#0110;                // 进料超时
                 #stat_wState := 90;
             END_IF;

         20: // 推料
             IF (#stat_aCylA.q_xBusy = FALSE AND #stat_xQualityNG = FALSE)
                OR (#stat_aCylB.q_xBusy = FALSE AND #stat_xQualityNG = TRUE) THEN
                 IF (#i_xCylA_ExtEnd AND NOT #stat_xQualityNG)
                    OR (#i_xCylB_ExtEnd AND #stat_xQualityNG) THEN
                     #stat_tHold(IN := TRUE, PT := #i_tHoldTime);
                     IF #stat_tHold.Q THEN
                         #stat_tHold(IN := FALSE, PT := #i_tHoldTime);
                         #stat_wState := 30;
                     END_IF;
                 END_IF;
             END_IF;

         30: // 回位
             IF (#i_xCylA_RetEnd AND NOT #stat_xQualityNG)
                OR (#i_xCylB_RetEnd AND #stat_xQualityNG) THEN
                 #stat_wState := 0;
             END_IF;

         90: // Fault
             #q_xFault := TRUE;
             IF #i_xReset AND NOT #stat_xResetOld THEN
                 #q_wFaultCode := 0;
                 #stat_wState := 0;
             END_IF;
     END_CASE;

     #stat_xResetOld := #i_xReset;
     #q_wState := INT_TO_WORD(#stat_wState);
     // 输出一律取自设备层命令（工艺层不自己造输出）
     #q_xConveyor := #stat_aConv1.q_xMotor;
     #q_xCylA_Extend := #stat_aCylA.q_xExtendCmd;
     #q_xCylB_Extend := #stat_aCylB.q_xExtendCmd;

     // 停止：任何模式下都不许继续动
     IF #i_xStop OR NOT #i_xSafetyOK THEN
         #q_xConveyor := FALSE;
         #q_xCylA_Extend := FALSE;
         #q_xCylB_Extend := FALSE;
     END_IF;

END_FUNCTION_BLOCK

DATA_BLOCK "DB_Sort1"
{ S7_Optimized_Access := 'TRUE' }
VERSION : 0.1
   "FB_SortStation"
BEGIN
END_DATA_BLOCK

```

## 2. 仿真用例（8 条）

每条开头先「回初始态」（所有输入归零 → Reset 上升沿 → 确认 state=0），否则上一条的件和故障会串场。

```js
const GW=process.env.TIALINK_URL||'http://127.0.0.1:8080/mcp';
async function call(tool,args){
  const r=await fetch(GW,{method:'POST',headers:{'Content-Type':'application/json; charset=utf-8',Accept:'application/json, text/event-stream'},
    body:JSON.stringify({jsonrpc:'2.0',id:1,method:'tools/call',params:{name:tool,arguments:args}})});
  const j=await r.json(); const t=((j.result||{}).content||[]).map(c=>c.text??'').join('\n');
  try{return JSON.parse(t);}catch{return {raw:t};}
}
const sleep=ms=>new Promise(r=>setTimeout(r,ms));
const n=v=>v===true?1:(v===false?0:v);
const w=(nm,v)=>call('tialink_sim_write',{op:'set-tag',params:{name:nm,value:v}});
async function rd(nm){const j=await call('tialink_sim_read',{op:'tag',params:{name:nm}});return n(j.value?.value);}
async function snap(){return {state:await rd('DB_Sort1.q_wState'),code:await rd('DB_Sort1.q_wFaultCode'),
  conv:await rd('DO_Conv1_Run'),a:await rd('DO_CylA_Extend'),b:await rd('DO_CylB_Extend')};}
const DI=['DI_Conv1_PartAtEntry','DI_Conv1_PartAtSort','DI_Hmi_Start','DI_Hmi_Stop','DI_Hmi_Reset',
  'DI_CylA_ExtEnd','DI_CylA_RetEnd','DI_CylB_ExtEnd','DI_CylB_RetEnd','DI_Qual_ResultOK'];
async function clean(){                 // 干净初始态：所有输入归零 + 一次复位沿 + 确认 state=0
  await w('DI_Sys_SafetyOK',true);
  await w('DI_Hmi_Stop',true); await sleep(200);
  for(const d of DI) await w(d,false);
  await w('DI_Hmi_Stop',false);
  await w('DI_Hmi_Reset',true); await sleep(250); await w('DI_Hmi_Reset',false); await sleep(250);
  return (await snap()).state;
}
const rows=[]; const R=(id,sc,exp,act,pass)=>rows.push({id,sc,exp,act,pass});

let pre=await clean();
R(0,'干净初始态（用例前置）','state=0',`state=${pre}`,pre===0);

// T1
await w('DI_Sys_SafetyOK',true); await w('DI_Hmi_Start',true); await sleep(600);
let s=await snap();
R(1,'SafetyOK=1 且 Start=1','state=10, 输送机=1',`state=${s.state}, conv=${s.conv}`, s.state===10&&s.conv===1);

// T2
await w('DI_Qual_ResultOK',true); await w('DI_Conv1_PartAtEntry',true); await sleep(400);
await w('DI_Conv1_PartAtEntry',false); await w('DI_Conv1_PartAtSort',true); await sleep(600);
s=await snap();
R(2,'入口光电→分拣位光电（质量 OK）','state=20, 气缸A=1, 气缸B=0',`state=${s.state}, A=${s.a}, B=${s.b}`, s.state===20&&s.a===1&&s.b===0);

// T3
await w('DI_CylA_ExtEnd',true); await sleep(400); const t3a=await snap();
await sleep(900); const t3b=await snap();
R(3,'气缸A 伸出到位','0.4s 仍 20；保持 800ms 后 30',`0.4s=${t3a.state}; 1.3s=${t3b.state}`, t3a.state===20&&t3b.state===30);

// T4（先把 Start 撤掉，避免回 Idle 后立刻重新启动）
await w('DI_Hmi_Start',false); await sleep(200);
await w('DI_CylA_ExtEnd',false); await w('DI_CylA_RetEnd',true); await sleep(700);
s=await snap();
R(4,'气缸A 缩回到位（Start 已撤）','state=0',`state=${s.state}`, s.state===0);

// T5 进料超时：干净态启动，不给任何件
await clean();
await w('DI_Hmi_Start',true); await sleep(600); const t5a=await snap();
await sleep(2200); const t5b=await snap();
await sleep(2500); const t5c=await snap();
R(5,'干净态启动后无进料','任一设备/流程超时 → 90，且码可解释',
  `0.6s=${t5a.state}; 2.8s=${t5b.state}/0x${(t5b.code||0).toString(16)}; 5.4s=${t5c.state}/0x${(t5c.code||0).toString(16)}`,
  t5c.state===90 && t5c.code!==0);

// T6 复位（故障态）
await w('DI_Hmi_Start',false); await w('DI_Hmi_Reset',true); await sleep(400); await w('DI_Hmi_Reset',false); await sleep(500);
s=await snap();
R(6,'故障态下 Reset 上升沿','state=0, faultCode=0',`state=${s.state}, code=0x${(s.code||0).toString(16)}`, s.state===0&&s.code===0);

// T7 运行中停机
await clean(); await w('DI_Sys_SafetyOK',true); await w('DI_Hmi_Start',true); await sleep(500);
const t7a=await snap();
await w('DI_Hmi_Stop',true); await sleep(400); const t7b=await snap();
R(7,'运行中 Stop=1','停机前后：conv 1→0',`before conv=${t7a.conv} state=${t7a.state}; after conv=${t7b.conv}`, t7a.conv===1&&t7b.conv===0);

console.log('| # | 场景 | 期望 | 实测 | 判定 |');
console.log('|---|---|---|---|---|');
for(const r of rows) console.log(`| ${r.id} | ${r.sc} | ${r.exp} | ${r.act} | ${r.pass?'PASS':'FAIL'} |`);
console.log('\nPASS',rows.filter(r=>r.pass).length,'/',rows.length);

```

## 3. 第 9 条：气缸伸出中停机

```js
const GW=process.env.TIALINK_URL||'http://127.0.0.1:8080/mcp';
async function call(t,a){const r=await fetch(GW,{method:'POST',headers:{'Content-Type':'application/json; charset=utf-8',Accept:'application/json, text/event-stream'},
 body:JSON.stringify({jsonrpc:'2.0',id:1,method:'tools/call',params:{name:t,arguments:a}})});
 const j=await r.json();const x=((j.result||{}).content||[]).map(c=>c.text??'').join('\n');try{return JSON.parse(x);}catch{return{raw:x};}}
const sleep=ms=>new Promise(r=>setTimeout(r,ms));
const n=v=>v===true?1:(v===false?0:v);
const w=(nm,v)=>call('tialink_sim_write',{op:'set-tag',params:{name:nm,value:v}});
async function rd(nm){const j=await call('tialink_sim_read',{op:'tag',params:{name:nm}});return n(j.value?.value);}
async function snap(){return{state:await rd('DB_Sort1.q_wState'),code:await rd('DB_Sort1.q_wFaultCode'),a:await rd('DO_CylA_Extend'),conv:await rd('DO_Conv1_Run')};}
const DI=['DI_Conv1_PartAtEntry','DI_Conv1_PartAtSort','DI_Hmi_Start','DI_Hmi_Stop','DI_Hmi_Reset','DI_CylA_ExtEnd','DI_CylA_RetEnd','DI_CylB_ExtEnd','DI_CylB_RetEnd','DI_Qual_ResultOK'];
async function clean(){await w('DI_Sys_SafetyOK',true);await w('DI_Hmi_Stop',true);await sleep(200);
 for(const d of DI)await w(d,false);await w('DI_Hmi_Stop',false);
 await w('DI_Hmi_Reset',true);await sleep(250);await w('DI_Hmi_Reset',false);await sleep(250);return (await snap()).state;}

await clean();
await w('DI_Hmi_Start',true); await w('DI_Qual_ResultOK',true);
await w('DI_Conv1_PartAtEntry',true); await sleep(350); await w('DI_Conv1_PartAtEntry',false);
await w('DI_Conv1_PartAtSort',true); await sleep(500);
const a=await snap();                       // 伸出中
await w('DI_Hmi_Stop',true); await sleep(500);
const b=await snap();                       // 停机 0.5s
await sleep(3000); const c=await snap();    // 停机 3.5s（原实现会在这里报 0x0201/0x0202）
await w('DI_Hmi_Stop',false); await sleep(700);
const d=await snap();                       // 恢复：应重新伸出

console.log('| # | 场景 | 期望 | 实测 | 判定 |');
console.log('|---|---|---|---|---|');
const P=(exp,act,ok)=>`| 8 | 气缸伸出中 Stop | ${exp} | ${act} | ${ok?'PASS':'FAIL'} |`;
console.log(P('输出=0，state≠90，码=0', `0.5s: A=${b.a}, state=${b.state}, code=0x${(b.code||0).toString(16)}`, b.a===0&&b.state!==90&&b.code===0));
console.log(P('3.5s 后仍无故障（撤许可不算超时）', `3.5s: A=${c.a}, state=${c.state}, code=0x${(c.code||0).toString(16)}`, c.state!==90&&c.code===0));
console.log(P('撤 Stop 后重新伸出', `1s 后: A=${d.a}, state=${d.state}`, d.a===1));
console.log(`\n伸出中基线：state=${a.state}, A=${a.a}`);

```
