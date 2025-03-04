# togokey table data (aggregate SPARQList)(togoID SPARQL 2)

## Parameters

* `togoKey`
  * default: hgnc
* `properties`
  * default: [{"propertyId": "refex_specific_high_expression", "categoryIds": ["v32_40", "v25_40"]}, {"propertyId": "uniprot_keywords_cellular_component","categoryIds": ["472"]}, {"propertyId": "uniprot_pdb_existence", "categoryIds": ["1"]}, {"propertyId": "uniprot_chembl_assay_existence", "categoryIds": ["1"]},{"propertyId": "refex_specific_low_expression"}, {"propertyId": "uniprot_phospho_site"}, {"propertyId": "uniprot_keywords_biological_process"}]
* `queryIds` togoKey 100個程度ずつ
  * default: ["4942","5344","6148", "6265","6344","6677","6735","10593","10718","10876"]

## `primaryIds`
```javascript
async ({togoKey, properties, queryIds})=>{
  let fetchReq = async (url, options, body) => {
    console.log(body); // debug
    console.log(url);  // debug
    if (body) options.body = body;
    return await fetch(url, options).then(res=>res.json());
  }

  let options = {
    method: 'POST',
    headers: {
      'Accept': 'application/json',
      'Content-Type': 'application/x-www-form-urlencoded'
    }
  }

  let togositeConfig = "https://raw.githubusercontent.com/dbcls/togosite/develop/config/togosite-human/properties.json";
  let sparqlSplitter = "https://integbio.jp/togosite/sparqlist/api/sparqlist_splitter";
  let togoidApi = "https://integbio.jp/togosite/sparqlist/api/togoid_route_sparql";
  let togositeConfigJson = await fetchReq(togositeConfig, {method: "get"});
  let idLimit = 2000; // split 判定
  
  // label 取得
  let labelApi = "https://integbio.jp/togosite/sparqlist/api/togokey_label";
  let togoIdToLabel = {};
  if (togoKey != "togovar") {
    togoIdToLabel = await fetchReq(labelApi, options, "togoKey=" + togoKey + "&queryIds=" + queryIds);
  }
  
  let queryProperties = JSON.parse(properties);
  let queryPropertyIds = queryProperties.map(d => d.propertyId);
  let togoIdArray = JSON.parse(queryIds);
  let tableData = {};
  for (let togoId of togoIdArray) {
    tableData[togoId] = [];
  }
  // togosite.config.json で上から
  for (let configSubject of togositeConfigJson) {
    for (let configProperty of configSubject.properties) {
      if (queryPropertyIds.includes(configProperty.propertyId)) { // クエリに Hit したら
      //  if (configProperty.primaryKey == "hp" || configProperty.primaryKey == "nando"　|| configProperty.primaryKey == "togovar") continue; // TogoID API alt. 未対応
        console.log(configProperty.propertyId); // debug
        
        // get 'togoKey' ID - 'primalyKey' ID list via TogoID API
        let idPair = [];
        if (togoKey != configProperty.primaryKey) idPair = await fetchReq(togoidApi, options, "source=" + togoKey + "&target=" + configProperty.primaryKey + "&ids=" + encodeURIComponent(togoIdArray.join(" ")));
        else idPair = togoIdArray.map(d => {return {source_id: d, target_id: d} });
        let togo2primary = {};
        for (let d of idPair) {
          if (!togo2primary[d.source_id]) togo2primary[d.source_id] = [];
          togo2primary[d.source_id].push(d.target_id);
        }

        // get attributes of 'primaryKey' Ids
        let primaryIds = Array.from(new Set(idPair.map(d=>d.target_id))).join(",");
        let categoryIdsParam = "";
        for (let queryProperty of queryProperties) {
          if (queryProperty.propertyId == configProperty.propertyId) {
            if (queryProperty.categoryIds) categoryIdsParam = "&categoryIds=" + queryProperty.categoryIds.join(",");
            break;
          }
        }
        let objectList = [];
        let body = "mode=objectList&queryIds=" + encodeURIComponent(primaryIds) + categoryIdsParam;
        if (primaryIds.length <= idLimit) {
          objectList = await fetchReq(configProperty.data, options, body);
        } else {
          body += "&sparqlet=" + encodeURIComponent(configProperty.data) + "&limit=" + idLimit;
          objectList = await fetchReq(sparqlSplitter, options, body);
        }

        // mapping to 'togoKey' ID
        let primaryId2attribute = {};
        for (let d of objectList) {
          if (!primaryId2attribute[d.id]) primaryId2attribute[d.id] = [];
          primaryId2attribute[d.id].push(d);
        }
        for (let togoId of Object.keys(togo2primary)) {
          let attributeList = [];
          for (let promaryId of togo2primary[togoId]) {
            if (primaryId2attribute[promaryId]) attributeList = attributeList.concat(primaryId2attribute[promaryId]);
          }
          if (attributeList[0]) { 
            tableData[togoId].push({
              propertyId: configProperty.propertyId,
              propertyLabel: configProperty.label,
              propertyKey: configProperty.primaryKey,
              attributes: attributeList
            })
          }
        }
      }
    }
  }
  // object to list
  return togoIdArray.map(togoId=>{
    let obj = { id: togoId };
    if (togoIdToLabel[togoId]) obj.label = togoIdToLabel[togoId];
    obj.properties = tableData[togoId];
    return obj;
  });
}
```

