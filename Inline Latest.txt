'use strict';
const functions = require('firebase-functions');
const {WebhookClient} = require('dialogflow-fulfillment');
const {Card, Suggestion} = require('dialogflow-fulfillment');
const axios = require('axios'); 
const admin = require('firebase-admin');
process.env.DEBUG = 'dialogflow:debug';
 
exports.dialogflowFirebaseFulfillment = functions.https.onRequest((request, response) => {
  const agent = new WebhookClient({ request, response });
  console.log('Dialogflow Request headers: ' + JSON.stringify(request.headers));
  console.log('Dialogflow Request body: ' + JSON.stringify(request.body));
  
  // Default Functions are configured here
  function welcome(agent) 
  {
    agent.add(`Greetings! Have a nice day. I'm an Insurance Assistant.\nHow Can I assist you?`);
    agent.add(new Suggestion("New Policy"));
    agent.add(new Suggestion("Renewal of Policy"));
  }
 
  function fallback(agent) {
    agent.add(`I didn't understand`);
    agent.add(`I'm sorry, can you try again?`);
  }
  
  // Extra Function for some extra functionalities
  function between(min, max) 
  {  
        return Math.floor(Math.random() * (max - min) + min);
  }
  
  // New Policy's Intents Handling methods are declared bellow
  function newpolicy(agent)
  {
  	agent.add(`That's great Can you please tell me what type of Insurance do you want ?`);
    //agent.add(new Suggestion("Life Insurance"));
    //agent.add(new Suggestion("Health Insurance"));
    agent.add(new Suggestion("Motor Insurance"));
  }
  
  function getModelName(agent) {
    var ManufacturerNm = agent.parameters.Manufacturers;
    return axios.get(`http://www.fijicare.com.fj/fijiwebapiuat/api/General/GetModel?Manufacturer_Nm=${ManufacturerNm}`)
    .then(response => {
      agent.add(`Which model of ${ManufacturerNm} are you using(e.g AXIO, A4, 1 SERIES, SUPERB etc..) ?`+JSON.stringify(response.data.ResponseObj.ModelObj.map(
        function(currentelement, index, arrayobj) 
      	{
            let Model_Nm = currentelement.Model_Nm;
     		return Model_Nm;
        }
      	), null, 2).replace('[','').replace(']','').replace(/"/g,'').replace(/,/g, ' '));
    }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  } 
  
  function newMotorInsuranceHandler(agent){
    // 'awaiting_modelname'context is used for accessing parameters of previous intent 'New Motor Insurance' intent
    const Modelcontext = agent.getContext('awaiting_modelname');
    const Name = Modelcontext.parameters.Name;
    const MotorManufacturer = Modelcontext.parameters.Manufacturers;
    const RegNum = Modelcontext.parameters.VehicleRegNum;
    // parameter from current intent
    var idv = agent.parameters.IDV;
    var modelName = agent.parameters.model_name;
    var Manufacturing_Year = agent.parameters.Manufacturing_Year;
    var EngineNumber = agent.parameters.EngineNum;
    var ChassisNumber = agent.parameters.ChassisNum;
    return axios.post('http://www.fijicare.com.fj/fijiwebapiuat/api/MobileApp/GetMotorQuotePremium',{Sum_Insured:`${idv}`, Registration_Yr: `${Manufacturing_Year}`, vehicle_Class:`${MotorManufacturer}`, Model_Nm: `${modelName}`, Registration_No:`${RegNum}`, Engine_No:`${EngineNumber}`, Chassis_No:`${ChassisNumber}`})
    .then(response => {
          var TotalPremium = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(tp => tp.Total_Premium)).replace('[','').replace(']','');
          agent.add(`${Name}, Your Quote premium for Motor Insurance is FJD ${TotalPremium} for 1 year.\nYour quote number is ${between(100000000,10000000000)}`);
    }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  }

  // Renew Policy's Intents Handling methods are declared bellow
  function Renewpolicy(agent)
  {
  	agent.add(`Happy to hear your interest. Can you please tell me which type of Insurance do you want to Renew ?`);
    //agent.add(new Suggestion("Renew Life Insurance"));
    //agent.add(new Suggestion("Health Insurance"));
    agent.add(new Suggestion("Renew Motor Insurance"));
  }
  function RenewMotorpolicy(agent)
  {
  	agent.add(`I can help you with following options on your policy.`);
    agent.add(new Suggestion("Policy Number"));
    agent.add(new Suggestion("Engine Number"));
    //agent.add(new Suggestion("Chassis Number"));
  }
  function RenewMotorPolicyWithPolicyNO(agent)
  {
    const PolicyNum = agent.parameters.PolicyNumber; 
    return axios.post('http://www.fijicare.com.fj/fijiwebapiuat/api/MobileApp/GetMotorQuoteRenewal',{Policy_No:`${PolicyNum}`})
    .then(response => {
     var startDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(st => st.Policy_Start_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
     var endDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(ed => ed.Policy_End_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
     var idv = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(si => si.Sum_Insured)).replace('[','').replace(']','');
     var coverNm = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(cnm => cnm.Cover_Nm)).replace('[','').replace(']','');
     var groossAmt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(gmt => gmt.Gross_Amt)).replace('[','').replace(']','');
     //Cover Array Creation and declaration
     var coverArr = ["Basic","WindScreen/Window","TPPD"];
     let BasicAmt = 0;
     let WindScreenAmt = 0;
     let TPPDAmt = 0;
     if(coverNm.replace(/"/g,'') == coverArr[0]){
     	BasicAmt = groossAmt;
     }
     else if(coverNm.replace(/"/g,'') == coverArr[1]){
     	WindScreenAmt = groossAmt;
     }
     else{
     	TPPDAmt = groossAmt;
     }
     
     var quoteNum = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(qt => qt.Quotation_No)).replace('[','').replace(']','').replace(/"/g,'');
     var renewPremium = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(rp => rp.Premium)).replace('[','').replace(']','');
     agent.add(`Policy Number: ${PolicyNum}\n`+`Policy Duration: ${startDt} to ${endDt}\n`+`IDV: $`+`${idv}\n`+`Cover:\n`+`\t >Basic : $${BasicAmt}\n\t >WindScreen/Window : $${WindScreenAmt}\n\t >TPPD : $${TPPDAmt}\n`+`Quotation number: ${quoteNum}\n`+`Renewal Policy Premium: $`+`${renewPremium}`);
     agent.add(new Suggestion("Pay Now"));
     agent.add(new Suggestion("Pay Later"));
     agent.add(new Suggestion("Change Cover Amount"));
    }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  }
  function RenewMotorPolicyWithEngineNO(agent)
  {
    const EngineNo = agent.parameters.EngineNum; 
    return axios.post('http://www.fijicare.com.fj/fijiwebapiuat/api/MobileApp/GetMotorQuoteRenewal',{Engine_No:`${EngineNo}`})
    .then(response => {
     var PolicyNum = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(st => st.Policy_No)).replace('[','').replace(']','').replace(/"/g,'');
     var startDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(st => st.Policy_Start_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
     var endDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(ed => ed.Policy_End_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
     var idv = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(si => si.Sum_Insured)).replace('[','').replace(']','');
     var coverNm = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(cnm => cnm.Cover_Nm)).replace('[','').replace(']','');
     var groossAmt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(gmt => gmt.Gross_Amt)).replace('[','').replace(']','');
     //Cover Array Creation and declaration
     var coverArr = ["Basic","WindScreen/Window","TPPD"];
     let BasicAmt = 0;
     let WindScreenAmt = 0;
     let TPPDAmt = 0;
     if(coverNm.replace(/"/g,'') == coverArr[0]){
     	BasicAmt = groossAmt;
     }
     else if(coverNm.replace(/"/g,'') == coverArr[1]){
     	WindScreenAmt = groossAmt;
     }
     else{
     	TPPDAmt = groossAmt;
     }
     
     var quoteNum = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(qt => qt.Quotation_No)).replace('[','').replace(']','').replace(/"/g,'');
     var renewPremium = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(rp => rp.Premium)).replace('[','').replace(']','');
     agent.add(`Policy Number: ${PolicyNum}\n`+`Policy Duration: ${startDt} to ${endDt}\n`+`IDV: $`+`${idv}\n`+`Cover:\n`+`\t >Basic : $${BasicAmt}\n\t >WindScreen/Window : $${WindScreenAmt}\n\t >TPPD : $${TPPDAmt}\n`+`Quotation number: ${quoteNum}\n`+`Renewal Policy Premium: $`+`${renewPremium}`);
     agent.add(new Suggestion("Pay Now"));
     agent.add(new Suggestion("Pay Later"));
     agent.add(new Suggestion("Change Cover Amount"));
    }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  }
  function CoverAmountChange(agent)
  {
  	const Modelcontext = agent.getContext('awaiting_policynumber');
    const PolicyNum = Modelcontext.parameters.PolicyNumber;
    return axios.post('http://www.fijicare.com.fj/fijiwebapiuat/api/MobileApp/GetMotorQuoteRenewal',{Policy_No:`${PolicyNum}`})
    .then(response => {
       var coverNm = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(cnm => cnm.Cover_Nm)).replace('[','').replace(']','');
       var groossAmt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(gmt => gmt.Gross_Amt)).replace('[','').replace(']','');
       //Cover Array Creation and declaration
       var coverArr = ["Basic","WindScreen/Window","TPPD"];
       let BasicAmt = 0;
       let WindScreenAmt = 0;
       let TPPDAmt = 0;
       if(coverNm.replace(/"/g,'') == coverArr[0]){
          BasicAmt = groossAmt;
       }
       else if(coverNm.replace(/"/g,'') == coverArr[1]){
          WindScreenAmt = groossAmt;
       }
       else{
          TPPDAmt = groossAmt;
       }
     	agent.add('Cover:');
      	agent.add(new Suggestion(`Basic: $`+`${BasicAmt}`));
      	agent.add(new Suggestion(`WindScreen/Window: $`+`${WindScreenAmt}`));
      	agent.add(new Suggestion(`TPPD: $`+`${TPPDAmt}`));
    }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  }
    function CoverAmountChangeEngineNum(agent)
  {
  	const Modelcontext = agent.getContext('awaiting_enginenumber');
    const EngineNo = Modelcontext.parameters.EngineNum;
    return axios.post('http://www.fijicare.com.fj/fijiwebapiuat/api/MobileApp/GetMotorQuoteRenewal',{Engine_No:`${EngineNo}`})
    .then(response => {
       var coverNm = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(cnm => cnm.Cover_Nm)).replace('[','').replace(']','');
       var groossAmt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(gmt => gmt.Gross_Amt)).replace('[','').replace(']','');
       //Cover Array Creation and declaration
       var coverArr = ["Basic","WindScreen/Window","TPPD"];
       let BasicAmt = 0;
       let WindScreenAmt = 0;
       let TPPDAmt = 0;
       if(coverNm.replace(/"/g,'') == coverArr[0]){
          BasicAmt = groossAmt;
       }
       else if(coverNm.replace(/"/g,'') == coverArr[1]){
          WindScreenAmt = groossAmt;
       }
       else{
          TPPDAmt = groossAmt;
       }
     	agent.add('Cover:');
      	agent.add(new Suggestion(`Basic: $`+`${BasicAmt}`));
      	agent.add(new Suggestion(`WindScreen/Window: $`+`${WindScreenAmt}`));
      	agent.add(new Suggestion(`TPPD: $`+`${TPPDAmt}`));
    }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  }
  function CoverAmountChangeBasic(agent)
  {
  	const Basic = agent.parameters.BasicAmt;
    const Modelcontext = agent.getContext('awaiting_policynumber');
    const PolicyNum = Modelcontext.parameters.PolicyNumber;
    return axios.post('http://www.fijicare.com.fj/fijiwebapiuat/api/MobileApp/GetMotorQuoteRenewal',{Policy_No:`${PolicyNum}`})
    .then(response => {
       var startDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(st => st.Policy_Start_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
       var endDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(ed => ed.Policy_End_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
       var idv = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(si => si.Sum_Insured)).replace('[','').replace(']','');
       var groossAmt = Basic;
       var quoteNum = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(qt => qt.Quotation_No)).replace('[','').replace(']','').replace(/"/g,'');
       var renewPremium = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(rp => rp.Premium)).replace('[','').replace(']','');
       agent.add(new Suggestion("Pay Now"));
       agent.add(new Suggestion("Pay Later"));
       agent.add(new Suggestion("Change Cover Amount"));
     }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  }
  function CoverAmountChangeBasicForEngineNum(agent)
  {
    const Basic = agent.parameters.BasicAmt;
  	const Modelcontext = agent.getContext('awaiting_enginenumber');
    const EngineNo = Modelcontext.parameters.EngineNum;
    return axios.post('http://www.fijicare.com.fj/fijiwebapiuat/api/MobileApp/GetMotorQuoteRenewal',{Engine_No:`${EngineNo}`})
    .then(response => {
       var startDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(st => st.Policy_Start_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
       var endDt = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(ed => ed.Policy_End_Dt)).replace('[','').replace(']','').replace(/"/g,'').slice(0,10);
       var idv = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(si => si.Sum_Insured)).replace('[','').replace(']','');
       var groossAmt = Basic;
       var quoteNum = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(qt => qt.Quotation_No)).replace('[','').replace(']','').replace(/"/g,'');
       var renewPremium = JSON.stringify(response.data.ResponseObj.MotorPoliciesObj.map(rp => rp.Premium)).replace('[','').replace(']','');
       agent.add(new Suggestion("Pay Now"));
       agent.add(new Suggestion("Pay Later"));
       agent.add(new Suggestion("Change Cover Amount"));
     }).catch(error => {agent.add(JSON.stringify(error));
    }); 
  }
  let intentMap = new Map();
  intentMap.set('Default Welcome Intent', welcome);
  intentMap.set('Default Fallback Intent', fallback);
  //New Policy Intents
  intentMap.set('1.New Policy', newpolicy);
  intentMap.set('1.2New Motor Insurance', getModelName);
  intentMap.set('1.2.1Vehicle Model Name', newMotorInsuranceHandler);
  //Renewal Policy Intents
  intentMap.set('2.Renewal Policy', Renewpolicy);
  intentMap.set('2.2Renew Motor Insurance', RenewMotorpolicy);
  intentMap.set('2.2.1Renew Motor Insurance_PolicyNumber',RenewMotorPolicyWithPolicyNO);
  intentMap.set('2.2.2Renew Motor Insurance_EngineNumber',RenewMotorPolicyWithEngineNO);
  intentMap.set('4.ChangeCoverAmount',CoverAmountChange);
  intentMap.set('5.ChangeCoverAmount_With_EngineNumber',CoverAmountChangeEngineNum);
  intentMap.set('4.1ChangeCoverAmount_Basic',CoverAmountChangeBasic);
  intentMap.set('5.1ChangeCoverAmount_EngineNum_Basic',CoverAmountChangeBasicForEngineNum);
  agent.handleRequest(intentMap);
});
