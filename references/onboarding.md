# Feast Onboarding

Guide for onboarding a new user to Feast.

## Overview

Onboarding creates a user profile that personalises the entire Feast experience. Take time to get this right — it's the foundation for everything.

## The Conversation

Keep it natural and friendly. Don't interrogate. Explain why each question matters.

### 1. Welcome & Explain

> "Let's set up Feast — your personal meal planning system. I'll ask a few questions to understand how you like to eat, then I can start planning meals that actually work for you."

### 2. Location (Required)

**Why:** Determines seasonality, units, available stores, cultural context.

> "Where are you based? Country is enough, but if you want hyper-local seasonal produce, I can note your region too."

Store as ISO country code (GB, US, DE, FR, AU, etc.).

### 3. Household Size (Required)

**Why:** Determines portions, shopping quantities.

> "How many people are you cooking for? Just yourself, or feeding others too?"

Also check for special cases:
- OMAD (one meal a day) — needs portion multiplier
- Batch cooking — cook once, portion twice

### 4. Week Structure (Required)

**Why:** Determines the planning and shopping cadence.

> "Let's talk about your week. When does your week feel like it starts — Sunday or Monday?"

> "How many days a week do you actually want to cook? Six? Seven? Less?"

> "Do you have a cheat day — a day you order in or eat out? Is it fixed or flexible?"

### 5. Dietary Requirements (Required)

**Why:** Safety first, health second. Be sensitive to religious constraints, halal, kosher etc.

> "Any allergies or dietary restrictions I absolutely need to know about? Things that would make you ill, not just things you don't fancy."

> "Are you in a particular dietary phase right now? Weight loss, maintenance, building muscle? Or just 'eating well'?"

> "Are you following a vegan or vegetarian diet?"

> "Does your faith restrict certain foods?" 

If weight loss/gain: ask about calorie targets (if they track) or just note the phase.

### 6. Cooking Setup (Important)

**Why:** Can't suggest slow cooker recipes if they don't have one.

> "What's your kitchen like? The usual oven and hob? Anything extra — air fryer, slow cooker, instant pot, fancy gadgets?"

> "How confident are you in the kitchen? Happy to try anything, or prefer foolproof recipes?"

> "How much time do you usually have for cooking? Quick weeknight meals, or happy to spend an hour?"

### 7. Preferences (Nice to Have)

**Why:** Makes recommendations better.

> "Spice — how hot do you like it? Be honest."

> "Any cuisines you particularly love? Or any you just don't enjoy?"

> "How adventurous are you? Stick to comfort food, or excited to try something completely new?"

> "Budget — strict limit, or just be sensible?"

> "Which supermarkets can you easily get to?"

### 8. Units (Required)

**Why:** Recipes must be readable.

> "Last one: how do you like your recipes? Celsius or Fahrenheit? Grams or ounces? Cups or millilitres?"

UK default: Celsius, metric, ml.  
US default: Fahrenheit, imperial, cups.

### 9. Notification Preferences (Required)

**Why:** Feast needs to remind users at the right times via the right channel.

> "How should I remind you about meal planning? I can use whatever channel you prefer — Telegram, Discord, or just here in the chat."

Ask about channel preference:
- `auto` — Use whatever channel they're currently talking on
- Specific channel — telegram, discord, signal, webchat, etc.

> "What times work for you? I'll need to ping you for:
> - Planning (usually Thursday evening)
> - Present and confirm the plan (Friday evening)
> - Shopping list (Firday night / Saturday morning)
> - Daily meal reveals (afternoon)
> - End of week review (Sunday evening)"

Get preferred times:
- Default to sensible times (18:00, 10:00, 15:00, 20:00)
- Adjust to their schedule
- Ask about quiet hours (when NOT to notify)

> "Any times I should never disturb you? Late night, early morning?"

Store in profile under `schedule.notifications` and `schedule.reminders`.

### 10. Set Up Cron Jobs

After saving the profile, create the recurring reminders using the cron tool.

#### Schedule Calculation

```python
# Calculate actual days based on weekStart
# If weekStart = "sunday":
#   planning = Thursday (day -3)
#   confirmation = Friday (day -2)
#   shopping = Saturday (day -1)
#   week starts Sunday (day 0)
#   review = Saturday (day 6)

# Example cron expressions (adjust times from profile):
# Planning: "0 18 * * 4" (Thursday 18:00)
# Confirmation: "0 18 * * 5" (Friday 18:00)
# Shopping: "0 10 * * 6" (Saturday 10:00)
# Daily reveals: Need one per cooking day
# Review: "0 20 * * 6" (Saturday 20:00 if week ends Sat)
```

#### Cron Job Configuration

**Important:** Use `agentTurn` payloads, not `systemEvent`. This ensures notifications are actually delivered to the user via their preferred channel.

Each cron job should be configured as:

```yaml
sessionTarget: "isolated"
payload:
  kind: "agentTurn"
  message: |
    Send a Feast notification.
    
    [Task-specific instructions — see individual jobs below]
    
    **Delivery Instructions:**
    1. Read the user profile at workspace/meals/profile.yaml
    2. Check schedule.notifications for their preference
    3. Compose the notification title and body text
    4. Deliver the notification:
    
       **If channel is "pushbullet":**
       Use the exec tool to run the pushbullet-notify script. IMPORTANT: To avoid
       shell quoting issues with apostrophes and special characters, write the body
       to a temporary file first, then use it:
       
       ```python
       import subprocess, tempfile, os
       body = "your composed message here"
       with tempfile.NamedTemporaryFile(mode='w', suffix='.txt', delete=False) as f:
           f.write(body)
           tmp = f.name
       # Read body from file to avoid shell escaping issues
       subprocess.run(['python3', 'skills/pushbullet-notify/scripts/send_notification.py',
                       '--title', '🍽️ Feast', '--body', body])
       os.unlink(tmp)
       ```
       
       Or simply pass the arguments directly via exec without shell interpolation —
       the key is to NEVER embed the body text inside a shell string with quotes.
       Use the exec tool's command as an array or use Python's subprocess with a list.
       
       **If channel is "telegram", "discord", or "signal":**
       Use the message tool with action=send and channel=[channel]
       
       **If channel is "webchat" or "auto":**
       Simply output the notification text — it will be delivered to the session
       
       **If push notifications are also enabled** (schedule.notifications.push):
       Send via push method in addition to primary channel.
       
       If the configured method fails, fall back to outputting the text directly.
    
    5. Confirm the notification was sent.
```

#### Reminder Messages

Each job type below includes its full agentTurn prompt. The prompts tell the spawned
agent what to do — including reading data files when the notification content should
be dynamic (e.g., morning hints, daily reveals).

1. **Planning reminder** (static message — no data lookup needed)
   ```yaml
   name: "Feast: Planning"
   schedule: { kind: "cron", expr: "0 18 * * 4", tz: "<user timezone>" }
   message: |
     Send a Feast notification.
     Body: "Time to plan next week's meals! Say 'let's plan meals' when you're ready."
     [delivery instructions]
   ```

2. **Confirmation reminder** (static)
   ```yaml
   name: "Feast: Confirmation"
   message: |
     Send a Feast notification.
     Body: "Your meal plan is ready for review. Say 'confirm meal plan' to see it and make any changes."
     [delivery instructions]
   ```

3. **Shopping list reminder** (static)
   ```yaml
   name: "Feast: Shopping List"
   message: |
     Send a Feast notification.
     Body: "Shopping list is ready! Say 'show shopping list' to review and plan your shop."
     [delivery instructions]
   ```

4. **Daily reveal** ⚡ DYNAMIC — reads the week plan to announce today's dish
   ```yaml
   name: "Feast: Daily Reveal"
   message: |
     You are sending today's Feast reveal notification — a short, exciting
     announcement of what's cooking tonight.

     **Steps:**
     1. Read the user profile at workspace/meals/profile.yaml
     2. Find the most recent week plan file in workspace/meals/weeks/
     3. Read the week plan and find TODAY's date entry
     4. If today is not a cooking day (cheat/skip), send a brief note and stop
     5. Send a notification that names the dish and its region of origin

     **The notification should:**
     - Name the dish
     - Name the region/country it's from
     - Be short and exciting (1-3 sentences)
     - Make them want to ask for the full reveal

     **Example:**
     "Tonight it's Salmon Teriyaki — straight from the kitchens of Osaka, Japan 🇯🇵
      Ask Lyra for the full reveal when you're ready to cook!"

     **Rules:**
     - Keep it SHORT — dish name, region, a spark of excitement
     - The full reveal (recipe, music, atmosphere) happens in the main session
       when the user asks for it
     - If no week plan exists or today has no meal, say so honestly

     [delivery instructions]
   ```

5. **Morning hint** ⚡ DYNAMIC — reads the week plan to craft a teaser
   ```yaml
   name: "Feast: Morning Hint"
   message: |
     You are sending a Feast morning hint — a cryptic teaser to build anticipation
     for today's meal.

     **Steps:**
     1. Read the user profile at workspace/meals/profile.yaml
     2. Find the most recent week plan file in workspace/meals/weeks/
     3. Find TODAY's date entry in the plan
     4. If today is not a cooking day (cheat/skip), skip — send nothing
     5. Craft a short, intriguing hint (2-3 sentences max) that:
        - Alludes to something from the PLACE (region, culture, a vivid detail)
          without naming the country
        - OR hints at the DISH in a playful/mysterious way without naming it
        - OR references something from currentContext, music, or cultural story
        - Feels personal and evocative, not generic or corporate
        - Builds genuine curiosity for the afternoon reveal
     6. Send the hint via the user's preferred notification channel

     **Good examples:**
     - "Tonight you're heading somewhere they greet each other by asking if
       they've eaten yet. The smoke rises, the neon glows... 🔥"
     - "A 17th century technique. A glossy coating. A city that ruined itself
       through food. Intrigued?"
     - "The band topping the charts for 1,500 straight days comes from nearby...
       but tonight's destination has its own rhythm. 🎵"

     **Rules:**
     - NEVER reveal the dish name or country directly
     - NEVER use generic filler like "cooking adventure awaits"
     - Keep it short — this is a teaser, not an essay
     - If no week plan exists or today has no meal, send nothing

     [delivery instructions]
   ```

6. **Week review** (static)
   ```yaml
   name: "Feast: Week Review"
   message: |
     Send a Feast notification.
     Body: "How was this week's cooking? Say 'review meals' to rate your dishes and capture what worked!"
     [delivery instructions]
   ```

**Store the cron job IDs** in `profile.schedule.cronJobs` so they can be updated or removed later.

### 11. Confirm & Save

Read back a summary:

> "Okay, let me make sure I've got this right..."

Then save to `workspace/meals/profile.yaml`.

## Post-Onboarding

### 1. Create the workspace structure

Create all necessary files and folders:
   ```
   workspace/meals/
   ├── profile.yaml
   ├── history.yaml (empty)
   ├── favourites.yaml (empty)
   ├── failures.yaml (empty)
   └── weeks/
   ```

### 2. Ensure reference files exist for user's location

Based on the user's country, check that the seasonality guide exists:
- UK → `references/seasonality/uk.md` (included)
- US → `references/seasonality/us.md` (create if needed)
- Other → Create basic guide or note as limitation

If the seasonality guide for their region doesn't exist, either:
1. Create a basic one during onboarding
2. Note that seasonal suggestions will be limited
3. Offer to research and create one later

The nutrition guide (`references/nutrition.md`) is universal and already included.

### 3. Create cron jobs for all reminders (see step 10).

3. **Confirm everything is set up:**
   > "All set! Here's what I've scheduled:
   > - Planning reminder: Thursdays at 6pm
   > - Plan confirmation: Fridays at 6pm
   > - Shopping list: Saturdays at 10am
   > - Daily reveals: 3pm on cooking days
   > - Week review: Saturday at 8pm
   >
   > I'll message you on [channel]. You can change any of this anytime."

4. **First plan:**
   > "Want to plan your first week now, or shall we wait for Thursday?"

## Managing Schedules Later

Users can adjust their schedule anytime:

- "Change my planning reminder to 7pm"
- "Turn off morning hints"
- "Move my reminders to Telegram"

When updating:
1. Remove old cron job(s) using stored ID
2. Create new cron job(s) with updated settings
3. Update profile with new job IDs

## Removing Feast

If a user wants to stop using Feast:
1. Remove all cron jobs using stored IDs
2. Optionally archive their data
3. Clean up workspace/meals/ folder
