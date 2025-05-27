function fetchEmailsForO2O47() {
  var sheet = SpreadsheetApp.getActiveSpreadsheet().getActiveSheet();
  var API_KEY = "e73cf99301d7894d233ae9e3a5c84c6b744240ea";
  var SEARCH_ENGINE_URL = "https://google.serper.dev/search";
  var startRow = 2;
  var endRow = 47;
  var sourceCol = "O";
  var targetCol = "P";
  var batchSize = 10;

  var companyNames = sheet.getRange(sourceCol + startRow + ":" + sourceCol + endRow)
                          .getValues().flat().filter(String);
  
  if (companyNames.length === 0) {
    Logger.log("Brak danych w zakresie O2:O47");
    return;
  }

  var queries = companyNames.map(company => ({ q: company + " email kontakt", gl: "pl", hl: "pl", device: "desktop" }));

  for (var i = 0; i < queries.length; i += batchSize) {
    var batch = queries.slice(i, i + batchSize);
    
    var response = UrlFetchApp.fetch(SEARCH_ENGINE_URL, {
      method: "post",
      headers: {
        "X-API-KEY": API_KEY,
        "Content-Type": "application/json"
      },
      payload: JSON.stringify(batch),
      muteHttpExceptions: true 
    });

    if (response.getResponseCode() !== 200) {
      Logger.log("Błąd API: " + response.getContentText());
      continue;
    }

    var results = JSON.parse(response.getContentText());

    var emails = results.map(extractEmail);

    var targetRange = sheet.getRange(targetCol + (startRow + i) + ":" + targetCol + (startRow + i + emails.length - 1));
    targetRange.setValues(emails.map(e => [e])); 
  }
}

function extractEmail(result) {
  if (!result || !result.organic || result.organic.length === 0) return ["Brak danych"];

  var emailRegex = /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g;
  
  for (var i = 0; i < result.organic.length; i++) {
    var snippet = result.organic[i].snippet;
    var link = result.organic[i].link;

    if (snippet) {
      var emails = snippet.match(emailRegex);
      if (emails) return [emails[0]];
    }
    
    if (link) {
      try {
        var pageResponse = UrlFetchApp.fetch(link, {
          muteHttpExceptions: true,
          followRedirects: true,
          headers: { "User-Agent": "Mozilla/5.0" } // Udajemy przeglądarkę
        });

        var pageText = pageResponse.getContentText();
        var emailsFromPage = pageText.match(emailRegex);
        if (emailsFromPage) return [emailsFromPage[0]];

        // Jeśli email ukryty za JavaScriptem – próbujemy znaleźć alternatywny sposób
        var advancedEmails = extractHiddenEmails(pageText);
        if (advancedEmails.length > 0) return [advancedEmails[0]];
      } catch (e) {
        Logger.log("Błąd pobierania strony: " + link);
      }
    }
  }
  return ["Nie znaleziono e-maila"];
}

function extractHiddenEmails(html) {
  var hiddenEmailRegex = /(?:mailto:|data-email=|data-contact=)["']?([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})["']?/g;
  var matches = [];
  var match;
  
  while ((match = hiddenEmailRegex.exec(html)) !== null) {
    matches.push(match[1]);
  }
  
  return matches;
}
