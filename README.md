# Google-Trends-Automated-Reporting-System
Google Trends Automated Reporting System (Pipedream + Scrapeless + Discord)
In the fields of digital marketing, SEO, and emerging trend analysis, staying up-to-date with changes in Google Trends is essential. However, manually checking and compiling keyword trend data is time-consuming, error-prone, and inefficient. To solve this, we’ve built an automated reporting system that integrates Pipedream, Scrapeless API, and Discord Webhook to streamline the entire process—from keyword collection and data processing to result delivery.

This article will walk you through the components and implementation of this automated system in detail.

## Key Features
- Automated Keyword Trend Analysis: Fetches trend data for the past month using Scrapeless.

- Smart Scoring & Processing: Automatically calculates average popularity and trend fluctuation for each keyword.

- Visual Analytical Reports: Generates structured reports with visuals and sends them directly to Discord.



---

## Prerequisites

1. **[Register on Scrapeless](https://app.scrapeless.com/passport/login?utm_source=official&utm_medium=blog&utm_campaign=pipedream)** and obtain your API Key.
2. **Locate your API Token** and copy it for later use.

![Locate your API Token](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/567cbcf7e4d9f594fc61fcb30cfe87e4.png)



⚠️ **Note:** Keep your API Token secure and **do not share it** with others.

---

3. **Log in to Pipedream**
4. Click **"+ New Workflow"** to create a new workflow.

![+ New Workflow](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/401c9b27d87d4dac2f052f57687aac98.png)


---

## How to Create a Google Trends Automated Reporting System (Pipedream + Scrapeless + Discord)

**Workflow Structure**

| Step Name        | Type           | Function                                                       |
|------------------|----------------|----------------------------------------------------------------|
| `trigger`        | Schedule       | Triggers the workflow on a schedule (e.g., every hour)         |
| `http`           | Node.js Code   | Submits scraping tasks to Scrapeless and fetches the results   |
| `code_step`      | Node.js Code   | Parses and processes the scraped data                          |
| `discord_notify` | Node.js Code   | Sends the analysis results to Discord via webhook              |

### Step 1: Scheduled Trigger
Select the trigger type as Schedule and set the trigger time to automatically execute this workflow every Sunday at UTC time 16:00.

![Select the trigger type](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/eef83f155941437208c47e18355fe309.png)



### Step 2 - http (request Scrapeless to crawl keyword trends)

![http (request Scrapeless to crawl keyword trends)](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/e4a18e88df28a127863586e05afc64f1.png)


1. Add a Node.js code step. The following is a code example integrating the Scrapeless API call logic

```
import { axios } from "@pipedream/platform"

export default defineComponent({
  async run({ steps, $ }) {
    const keywords = ["AI", "Machine Learning", "Data Science"];
    const allResults = {};
    
    for (const keyword of keywords) {
      console.log(`Processing keyword: ${keyword}`);
      
      const payload = {
        actor: "scraper.google.trends",
        input: {
          q: keyword,
          date: "today 1-m",
          data_type: "interest_over_time",
          hl: "en",
          tz: "420",
          geo: "",
          cat: "",
          property: "",
        },
        proxy: {
          country: "",
        }
      };

      try {
        const response = await axios($, {
          method: 'POST',
          url: 'https://api.scrapeless.com/api/v1/scraper/request',
          headers: {
            'Content-Type': 'application/json',
            'x-api-token': 'Scrapeless API KEY'
          },
          data: payload
        });
        
        console.log(`Response for ${keyword}:`, response);
        
        if (response.job_id) {
          console.log(`Job created for ${keyword}, ID: ${response.job_id}`);
          
          let attempts = 0;
          const maxAttempts = 12;
          let jobResult = null;
          
          while (attempts < maxAttempts) {
            await new Promise(resolve => setTimeout(resolve, 10000));
            attempts++;
            
            try {
              const resultResponse = await axios($, {
                method: 'GET',
                url: `https://api.scrapeless.com/api/v1/scraper/result/${response.job_id}`,
                headers: {
                  'x-api-token': 'Scrapeless API KEY'
                }
              });
              
              console.log(`Attempt ${attempts} for ${keyword}:`, resultResponse);
              
              if (resultResponse.status === 'completed') {
                jobResult = resultResponse.result;
                console.log(`Job completed for ${keyword}:`, jobResult);
                break;
              } else if (resultResponse.status === 'failed') {
                console.error(`Job failed for ${keyword}:`, resultResponse);
                break;
              } else {
                console.log(`Job still running for ${keyword}, status: ${resultResponse.status}`);
              }
            } catch (error) {
              console.error(`Error checking job status for ${keyword}:`, error);
            }
          }
          
          if (jobResult) {
            allResults[keyword] = jobResult;
          } else {
            allResults[keyword] = { 
              error: `Job timeout or failed after ${attempts} attempts`,
              job_id: response.job_id 
            };
          }
          
        } else {
          allResults[keyword] = response;
        }
        
      } catch (error) {
        console.error(`Error for ${keyword}:`, error);
        allResults[keyword] = { 
          error: `Request failed: ${error.message}` 
        };
      }
    }
    
    console.log("Final results:", JSON.stringify(allResults, null, 2));
    return allResults;
  },
})
```


---

**Note:**

* This step uses Scrapeless's `scraper.google.trends` actor to retrieve the **"interest\_over\_time"** data for each keyword.
* The default keywords are: `["AI", "Machine Learning", "Data Science"]`. You can replace them as needed.
* Replace `YOUR_API_TOKEN` with your actual API token.


### Step 3 - code_step (processing crawled data)

![Step 3 - code_step (processing crawled data)](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/3189262641136969fc0269c9f6e6cc8b.png)

1. Add another Node.js code step. Here is the code example

```
export default defineComponent({
  async run({ steps, $ }) {
    const httpData = steps.http.$return_value;
    
    console.log("Received HTTP data:", JSON.stringify(httpData, null, 2));
    
    const processedData = {};
    const timestamp = new Date().toISOString();
    
    for (const [keyword, data] of Object.entries(httpData)) {
      console.log(`Processing ${keyword}:`, data);
      
      let timelineData = null;
      let isPartial = false;
      
      if (data?.interest_over_time?.timeline_data) {
        timelineData = data.interest_over_time.timeline_data;
        isPartial = data.interest_over_time.isPartial || false;
      } else if (data?.timeline_data) {
        timelineData = data.timeline_data;
      } else if (data?.interest_over_time && Array.isArray(data.interest_over_time)) {
        timelineData = data.interest_over_time;
      }
      
      if (timelineData && Array.isArray(timelineData) && timelineData.length > 0) {
        console.log(`Found ${timelineData.length} data points for ${keyword}`);
        
        const values = timelineData.map(item => {
          const value = item.value || item.interest || item.score || item.y || 0;
          return parseInt(value) || 0;
        });
        
        const validValues = values.filter(v => v >= 0);
        
        if (validValues.length > 0) {
          const average = Math.round(validValues.reduce((sum, val) => sum + val, 0) / validValues.length);
          const trend = validValues.length >= 2 ? validValues[validValues.length - 1] - validValues[0] : 0;
          const max = Math.max(...validValues);
          const min = Math.min(...validValues);
          
          const recentValues = validValues.slice(-7);
          const earlyValues = validValues.slice(0, 7);
          const recentAvg = recentValues.length > 0 ? recentValues.reduce((a, b) => a + b, 0) / recentValues.length : 0;
          const earlyAvg = earlyValues.length > 0 ? earlyValues.reduce((a, b) => a + b, 0) / earlyValues.length : 0;
          const weeklyTrend = Math.round(recentAvg - earlyAvg);
          
          processedData[keyword] = {
            keyword,
            average,
            trend,
            weeklyTrend,
            values: validValues,
            max,
            min,
            dataPoints: validValues.length,
            isPartial,
            timestamps: timelineData.map(item => item.time || item.date || item.timestamp),
            lastValue: validValues[validValues.length - 1],
            firstValue: validValues[0],
            volatility: Math.round(Math.sqrt(validValues.map(x => Math.pow(x - average, 2)).reduce((a, b) => a + b, 0) / validValues.length)),
            status: 'success'
          };
        } else {
          processedData[keyword] = {
            keyword,
            average: 0,
            trend: 0,
            weeklyTrend: 0,
            values: [],
            max: 0,
            min: 0,
            dataPoints: 0,
            isPartial: false,
            error: "No valid values found in timeline data",
            status: 'error',
            rawTimelineLength: timelineData.length
          };
        }
      } else {
        processedData[keyword] = {
          keyword,
          average: 0,
          trend: 0,
          weeklyTrend: 0,
          values: [],
          max: 0,
          min: 0,
          dataPoints: 0,
          isPartial: false,
          error: "No timeline data found",
          status: 'error',
          availableKeys: data ? Object.keys(data) : []
        };
      }
    }
    
    const summary = {
      timestamp,
      totalKeywords: Object.keys(processedData).length,
      successfulKeywords: Object.values(processedData).filter(d => d.status === 'success').length,
      period: "today 1-m",
      data: processedData
    };
    
    console.log("Final processed data:", JSON.stringify(summary, null, 2));
    return summary;
  },
})

```

---

It will calculate the following metrics based on the data returned by Scrapeless:

* **Average value**
* **Weekly trend change** (last 7 days vs previous 7 days)
* **Maximum / Minimum values**
* **Volatility**

**Note:**

* Each keyword will generate a detailed analysis object, making it easier for visualization and further notifications.

### Step 4 - discord_notify (Push analysis report to Discord)

![Step 4 - discord_notify (Push analysis report to Discord)](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/f18726b4fe24fa4e2815d32624288b79.png)

1. Add the last Node.js step, the following is the code example

```
import { axios } from "@pipedream/platform"

export default defineComponent({
  async run({ steps, $ }) {
    const processedData = steps.Code_step.$return_value;
    
    const discordWebhookUrl = "https://discord.com/api/webhooks/1380448411299614821/MXzmQ14TOPK912lWhle_7qna2VQJBjWrdCkmHjdEloHKhYXw0fpBrp-0FS4MDpDB8tGh";
    
    console.log("Processing data for Discord:", JSON.stringify(processedData, null, 2));
    
    const currentDate = new Date().toLocaleString('en-US');
    
    const keywords = Object.values(processedData.data);
    const successful = keywords.filter(k => k.status === 'success');
    
    let bestPerformer = null;
    let worstPerformer = null;
    let strongestTrend = null;
    
    if (successful.length > 0) {
      bestPerformer = successful.reduce((max, curr) => curr.average > max.average ? curr : max);
      worstPerformer = successful.reduce((min, curr) => curr.average < min.average ? curr : min);
      strongestTrend = successful.reduce((max, curr) => Math.abs(curr.weeklyTrend) > Math.abs(max.weeklyTrend) ? curr : max);
    }
    
    const discordMessage = {
      content: `📊 **Google Trends Report** - ${currentDate}`,
      embeds: [
        {
          title: "🔍 Trends Analysis",
          color: 3447003,
          timestamp: new Date().toISOString(),
          fields: [
            {
              name: "📈 Summary",
              value: `**Period:** Last month\n**Keywords analyzed:** ${processedData.totalKeywords}\n**Success:** ${processedData.successfulKeywords}/${processedData.totalKeywords}`,
              inline: false
            }
          ]
        }
      ]
    };
    
    if (successful.length > 0) {
      const keywordDetails = successful.map(data => {
        const trendIcon = data.weeklyTrend > 5 ? '🚀' : 
                         data.weeklyTrend > 0 ? '📈' : 
                         data.weeklyTrend < -5 ? '📉' : 
                         data.weeklyTrend < 0 ? '⬇️' : '➡️';
        
        const performanceIcon = data.average > 70 ? '🔥' : 
                               data.average > 40 ? '✅' : 
                               data.average > 20 ? '🟡' : '🔴';
        
        return {
          name: `${performanceIcon} ${data.keyword}`,
          value: `**Score:** ${data.average}/100\n**Trend:** ${trendIcon} ${data.weeklyTrend > 0 ? '+' : ''}${data.weeklyTrend}\n**Range:** ${data.min}-${data.max}`,
          inline: true
        };
      });
      
      discordMessage.embeds[0].fields.push(...keywordDetails);
    }
    
    if (bestPerformer && successful.length > 1) {
      discordMessage.embeds.push({
        title: "🏆 Key Highlights",
        color: 15844367,
        fields: [
          {
            name: "🥇 Top Performer",
            value: `**${bestPerformer.keyword}** (${bestPerformer.average}/100)`,
            inline: true
          },
          {
            name: "📊 Lowest Score",
            value: `**${worstPerformer.keyword}** (${worstPerformer.average}/100)`,
            inline: true
          },
          {
            name: "⚡ Strongest Trend",
            value: `**${strongestTrend.keyword}** (${strongestTrend.weeklyTrend > 0 ? '+' : ''}${strongestTrend.weeklyTrend})`,
            inline: true
          }
        ]
      });
    }
    
    const failed = keywords.filter(k => k.status === 'error');
    if (failed.length > 0) {
      discordMessage.embeds.push({
        title: "❌ Errors",
        color: 15158332,
        fields: failed.map(data => ({
          name: data.keyword,
          value: data.error || "Unknown error",
          inline: true
        }))
      });
    }
    
    console.log("Discord message to send:", JSON.stringify(discordMessage, null, 2));
    
    try {
      const response = await axios($, {
        method: 'POST',
        url: discordWebhookUrl,
        headers: {
          'Content-Type': 'application/json'
        },
        data: discordMessage
      });
      
      console.log("Discord webhook response:", response);
      
      return {
        webhook_sent: true,
        platform: "discord",
        message_sent: true,
        keywords_analyzed: processedData.totalKeywords,
        successful_keywords: processedData.successfulKeywords,
        timestamp: currentDate,
        discord_response: response
      };
      
    } catch (error) {
      console.error("Discord webhook error:", error);
      
      const simpleMessage = {
        content: `📊 **Google Trends Report - ${currentDate}**\n\n` +
                `📈 **Results (${processedData.successfulKeywords}/${processedData.totalKeywords}):**\n` +
                successful.map(data => 
                  `• **${data.keyword}**: ${data.average}/100 (${data.weeklyTrend > 0 ? '+' : ''}${data.weeklyTrend})`
                ).join('\n') +
                (failed.length > 0 ? `\n\n❌ **Errors:** ${failed.map(d => d.keyword).join(', ')}` : '')
      };
      
      try {
        const fallbackResponse = await axios($, {
          method: 'POST',
          url: discordWebhookUrl,
          headers: {
            'Content-Type': 'application/json'
          },
          data: simpleMessage
        });
        
        return {
          webhook_sent: true,
          platform: "discord",
          message_sent: true,
          fallback_used: true,
          discord_response: fallbackResponse
        };
        
      } catch (fallbackError) {
        console.error("Discord fallback error:", fallbackError);
        
        return {
          webhook_sent: false,
          platform: "discord",
          error: error.message,
          fallback_error: fallbackError.message,
          data_summary: {
            keywords: processedData.totalKeywords,
            successful: processedData.successfulKeywords,
            best_performer: bestPerformer?.keyword,
            best_score: bestPerformer?.average
          }
        };
      }
    }
  },
})
```
2. Replace YOUR DISCORD WEBHOOK URL with your own Webhook address
3. Click Deploy to run your workflow and get real-time information.

### Step 5 - Receive real-time Google Trends information

You can receive data for checking keywords directly in Discord:


![Step 5 - Receive real-time Google Trends information](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/7169818a2a9ab050b812d04b40d5c480.png)

**The following is the complete link diagram：**

![The following is the complete link diagram](https://assets.scrapeless.com/prod/posts/build-google-trends-monitor-with-pipedream/f75d04384f0a80200a0509310fd14680.png)



> **✅ Live Now: Scrapeless Official Integration on Pipedream**
> 
> Scrapeless is now officially available on Pipedream’s integration hub! With just a few clicks, you can call our powerful Google Trends API directly from your Pipedream workflows—no setup, no servers required.
> 
> Whether you’re building real-time dashboards, automating marketing intelligence, or powering custom analytics, this integration gives you the fastest path to production-grade trend monitoring.
> 
> 👉 Start building instantly: **pipedream.com/apps/scrapeless**
 Drag, drop, and deploy your next trend-powered workflow—today.
 
Perfect for developers, analysts, and growth teams who need actionable insights, fast.

## Summary
Through the powerful combination of Pipedream, Scrapeless API and Discord, we built a Google Trends reporting system that does not require manual intervention and is automatically executed daily. This not only greatly improves work efficiency, but also makes marketing decisions more data-based.

If you also need to build a similar data automation analysis system, you might as well try this technology combination!

---
Scrapeless operates in full compliance with applicable laws and regulations, accessing only publicly available data in accordance with platform terms of service. This solution is designed for legitimate business intelligence and research purposes.



