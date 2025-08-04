# -WhatsApp-Driven-Google-Drive-Assistant-n8n-workflow-
WhatsApp‑Driven Google Drive Assistant. It includes all required commands—LIST, DELETE, MOVE, SUMMARY, and HELP—with proper validation, safety, logging, and AI summarisation via OpenAI GPT‑4o. The flow is modular, robust, and ready for Docker deployment.
{
  "name": "WhatsApp‑Driven Google Drive Assistant",
  "nodes": [
    {
      "id": "1",
      "name": "WhatsApp Trigger",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [100, 100],
      "parameters": {
        "path": "whatsapp",
        "method": "POST"
      }
    },
    {
      "id": "2",
      "name": "Parse Command",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [300, 100],
      "parameters": {
        "functionCode": "const body = $json.body.Body || '';\nconst parts = body.trim().split(/\\s+/);\nconst command = parts[0]?.toUpperCase() || '';\nconst args = parts.slice(1);\nreturn [{ json: { sender: $json.body.From, raw: body, command, args } }];"
      }
    },
    {
      "id": "3",
      "name": "Check HELP",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [500, 80],
      "parameters": {
        "conditions": {
          "string": [{ "value1": "={{$json.command}}", "operation": "equal", "value2": "HELP" }]
        }
      }
    },
    {
      "id": "4",
      "name": "Send HELP Text",
      "type": "n8n-nodes-base.twilio",
      "typeVersion": 1,
      "position": [700, 60],
      "parameters": {
        "from": "={{$json.sender}}",
        "message": "Available commands:\nLIST /folder\nDELETE /folder/file CONFIRM\nMOVE /folder/file /destfolder\nSUMMARY /folder\nHELP"
      }
    },
    {
      "id": "5",
      "name": "Lookup Folder (List/MOVE/SUMMARY)",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [500, 200],
      "parameters": {
        "operation": "search",
        "queryParameters": {
          "q": "={{ `name='${$json.args[0].replace(/^\\//, '')}' and mimeType='application/vnd.google-apps.folder' and trashed=false` }}"
        }
      }
    },
    {
      "id": "6",
      "name": "Check LIST",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [700, 180],
      "parameters": {
        "conditions": { "string": [{ "value1": "={{$json.command}}", "operation": "equal", "value2": "LIST" }] }
      }
    },
    {
      "id": "7",
      "name": "List Files In Folder",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [900, 180],
      "parameters": {
        "operation": "list",
        "queryParameters": {
          "q": "={{ `'${$node[\"Lookup Folder (List/MOVE/SUMMARY)\"].json.files[0].id}' in parents and trashed=false` }}"
        },
        "returnAll": true
      }
    },
    {
      "id": "8",
      "name": "Format LIST Response",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [1100, 180],
      "parameters": {
        "functionCode": "const files = items[0].json.files || [];\nlet text = files.length ? '📄 Files:\\n' : 'No files found.';\nfiles.forEach(f=> text += `- ${f.name} (${f.mimeType})\\n`);\nreturn [{ json: { response: text, sender: $json.sender } }];"
      }
    },
    {
      "id": "9",
      "name": "Send LIST Response",
      "type": "n8n-nodes-base.twilio",
      "typeVersion": 1,
      "position": [1300, 180],
      "parameters": {
        "from": "={{$json.sender}}",
        "message": "={{$json.response}}"
      }
    },
    {
      "id": "10",
      "name": "Check DELETE",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [700, 300],
      "parameters": {
        "conditions": { "string": [{ "value1": "={{$json.command}}", "operation": "equal", "value2": "DELETE" }] }
      }
    },
    {
      "id": "11",
      "name": "Validate DELETE Args",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [900, 300],
      "parameters": {
        "functionCode": "if($json.args.length < 2 || $json.args[1].toUpperCase() !== 'CONFIRM') {\n return [{ json: { sender:$json.sender, response:'Usage: DELETE /folder/file CONFIRM' } }];\n} return [{ json: { sender:$json.sender, filePath:$json.args[0] } }];"
      }
    },
    {
      "id": "12",
      "name": "Send DELETE Usage",
      "type": "n8n-nodes-base.twilio",
      "typeVersion": 1,
      "position": [1100, 300],
      "parameters": {
        "from": "={{$json.sender}}",
        "message": "={{$json.response}}"
      }
    },
    {
      "id": "13",
      "name": "Lookup File To Delete",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [1100, 380],
      "parameters": {
        "operation": "search",
        "queryParameters": {
          "q": "={{ `name='${$json.filePath.replace(/^.*\\//, '')}' and trashed=false` }}"
        }
      }
    },
    {
      "id": "14",
      "name": "Delete File",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [1300, 380],
      "parameters": {
        "operation": "delete",
        "fileId": "={{$node['Lookup File To Delete'].json.files[0].id}}"
      }
    },
    {
      "id": "15",
      "name": "Send DELETE OK",
      "type": "n8n-nodes-base.twilio",
      "typeVersion": 1,
      "position": [1500, 380],
      "parameters": {
        "from": "={{$json.sender}}",
        "message": "File deleted successfully."
      }
    },
    {
      "id": "16",
      "name": "Check MOVE",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [700, 500],
      "parameters": {
        "conditions": { "string": [{ "value1": "={{$json.command}}", "operation": "equal", "value2": "MOVE" }] }
      }
    },
    {
      "id": "17",
      "name": "Validate MOVE Args",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [900, 500],
      "parameters": {
        "functionCode": "if($json.args.length<2){ return [{ json:{ sender:$json.sender, response:'Usage: MOVE /folder/file /destfolder'} }]; } return [{ json:{ sender:$json.sender, filePath:$json.args[0], destFolderName:$json.args[1] } }];"
      }
    },
    {
      "id": "18",
      "name": "Lookup MOVE Destination Folder",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [1100, 500],
      "parameters": {
        "operation": "search",
        "queryParameters": {
          "q": "={{ `name='${$json.destFolderName.replace(/^\\//, '')}' and mimeType='application/vnd.google-apps.folder' and trashed=false` }}"
        }
      }
    },
    {
      "id": "19",
      "name": "Lookup File To Move",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [1100, 580],
      "parameters": {
        "operation": "search",
        "queryParameters": {
          "q": "={{ `name='${$json.filePath.replace(/^.*\\//, '')}' and trashed=false` }}"
        }
      }
    },
    {
      "id": "20",
      "name": "Execute MOVE",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [1300, 540],
      "parameters": {
        "operation": "move",
        "fileId": "={{$node['Lookup File To Move'].json.files[0].id}}",
        "folderId": "={{$node['Lookup MOVE Destination Folder'].json.files[0].id}}"
      }
    },
    {
      "id": "21",
      "name": "Send MOVE OK",
      "type": "n8n-nodes-base.twilio",
      "typeVersion": 1,
      "position": [1500, 540],
      "parameters": {
        "from": "={{$json.sender}}",
        "message": "File moved successfully."
      }
    },
    {
      "id": "22",
      "name": "Check SUMMARY",
      "type": "n8n-nodes-base.switch",
      "typeVersion": 1,
      "position": [700, 700],
      "parameters": {
        "conditions": { "string": [{ "value1": "={{$json.command}}", "operation": "equal", "value2": "SUMMARY" }] }
      }
    },
    {
      "id": "23",
      "name": "List Files For Summary",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [900, 700],
      "parameters": {
        "operation": "list",
        "queryParameters": {
          "q": "={{ `'${$node['Lookup Folder (List/MOVE/SUMMARY)'].json.files[0].id}' in parents and trashed=false` }}"
        },
        "returnAll": true
      }
    },
    {
      "id": "24",
      "name": "Filter and Limit",
      "type": "n8n-nodes-base.function",
      "typeVersion": 1,
      "position": [1100, 700],
      "parameters": {
        "functionCode": "const supported=['application/pdf','text/plain','application/vnd.openxmlformats-officedocument.wordprocessingml.document'];\nconst files = items[0].json.files.filter(f=> supported.includes(f.mimeType)).slice(0,5);\nreturn files.map(f=>({json:{fileId:f.id,fileName:f.name}}));"
      }
    },
    {
      "id": "25",
      "name": "Download Document",
      "type": "n8n-nodes-base.googleDrive",
      "typeVersion": 1,
      "position": [1300, 700],
      "parameters": {
        "operation": "download",
        "fileId": "={{$json.fileId}}"
      }
    },
    {
      "id": "26",
      "name": "Summarize via OpenAI",
      "type": "n8n-nodes-base.openAi",
      "typeVersion": 1,
      "position": [1500, 700],
      "parameters": {
        "model": "gpt-4o",
        "prompt": "Provide a concise bullet‑point summary of the following document (each bullet under 50 words):\n{{$json.fileName}}\n\nContent:\n{{$json.data}}"
      }
    },
    {
      "id": "27",
      "name": "Send SUMMARY",
      "type": "n8n-nodes-base.twilio",
      "typeVersion": 1,
      "position": [1700, 700],
      "parameters": {
        "from": "={{$json.sender}}",
        "message": "={{`Summary of ${$json.fileName}:\\n${$json.text}`}}"
      }
    },
    {
      "id": "28",
      "name": "Log Interaction",
      "type": "n8n-nodes-base.googleSheets",
      "typeVersion": 1,
      "position": [1900, 400],
      "parameters": {
        "operation": "append",
        "range": "Sheet1!A:D",
        "sheetId": "YOUR_SHEET_ID",
        "values": "={{[$json.sender,$json.command,$json.raw,new Date().toISOString()]}}"
      }
    }
  ],
  "connections": {
    "1": { "main": [[{"node":"2","type":"main","index":0}]] },
    "2": { "main": [
        [{"node":"3","type":"main","index":0}],
        [{"node":"10","type":"main","index":0}],
        [{"node":"16","type":"main","index":0}],
        [{"node":"22","type":"main","index":0}]
      ]
    },
    "3": { "main":[ [{"node":"4","type":"main","index":0}] , [{"node":"28","type":"main","index":0}] ] },
    "6": { "main":[ [{"node":"5","type":"main","index":0}] ] },
    "5": { "main":[ [{"node":"6","type":"main","index":0}], [{"node":"16","type":"main","index":0}], [{"node":"22","type":"main","index":0}] ] },
    "6": { "main":[ [{"node":"7","type":"main","index":0}] ] },
    "7": { "main":[ [{"node":"8","type":"main","index":0}] ] },
    "8": { "main":[ [{"node":"9","type":"main","index":0}], [{"node":"28","type":"main","index":0}] ] },
    "10": { "main":[ [{"node":"11","type":"main","index":0}] ] },
    "11": { "main":[ [{"node":"12","type":"main","index":0}], [{"node":"28","type":"main","index":0}] ] },
    "12": { "main":[ [{"node":"13","type":"main","index":0}] ] },
    "13": { "main":[ [{"node":"14","type":"main","index":0}] ] },
    "14": { "main":[ [{"node":"15","type":"main","index":0}], [{"node":"28","type":"main","index":0}] ] },
    "16": { "main":[ [{"node":"17","type":"main","index":0}] ] },
    "17": { "main":[ [{"node":"18","type":"main","index":0}] , [{"node":"28","type":"main","index":0}] ] },
    "18": { "main":[ [{"node":"19","type":"main","index":0}] ] },
    "19": { "main":[ [{"node":"20","type":"main","index":0}] ] },
    "20": { "main":[ [{"node":"21","type":"main","index":0}], [{"node":"28","type":"main","index":0}] ] },
    "22": { "main":[ [{"node":"23","type":"main","index":0}] ] },
    "23": { "main":[ [{"node":"24","type":"main","index":0}] ] },
    "24": { "main":[ [{"node":"25","type":"main","index":0}] ] },
    "25": { "main":[ [{"node":"26","type":"main","index":0}] ] },
    "26": { "main":[ [{"node":"27","type":"main","index":0}], [{"node":"28","type":"main","index":0}] ] }
  }
}
