## AutoMail.toc
```toc
## Interface: 120001
## Title: AutoMail
## Notes: Adds a multi-line input and buttons to the Send Mail window to automatically attach specific items.
## Author: Zenithas
## Version: 1.1
## SavedVariables: AutoMailDB

AutoMail.lua
```

## AutoMail.lua
```lua
local addonName, addonTable = ...

-- Ensure our SavedVariable exists when the addon loads
local eventFrame = CreateFrame("Frame")
eventFrame:RegisterEvent("ADDON_LOADED")
eventFrame:RegisterEvent("MAIL_SHOW")

-- Variables to hold our UI elements
local configFrame
local attachQueue = {}
local isAttaching = false

-------------------------------------------------
-- Helper: Find an empty bag slot
-------------------------------------------------
local function FindEmptyBagSlot()
    for b = 0, 4 do
        for s = 1, C_Container.GetContainerNumSlots(b) do
            if not C_Container.GetContainerItemInfo(b, s) then
                return b, s
            end
        end
    end
    return nil, nil
end

-------------------------------------------------
-- Queue Processor for Attaching Items Safely
-------------------------------------------------
local function ProcessAttachQueue()
    if #attachQueue == 0 then
        isAttaching = false
        return
    end

    -- Check we have a free mail slot at all
    local hasFreeSlot = false
    for i = 1, 12 do
        if not GetSendMailItem(i) then 
            hasFreeSlot = true
            break 
        end
    end
    
    if not hasFreeSlot then 
        print("|cFF00FF00[AutoMail]|r: Mail slots are full!")
        attachQueue = {}
        isAttaching = false
        ClearCursor()
        return
    end

    local task = attachQueue[1]
    ClearCursor()
    
    -- SCENARIO A: We need the whole stack
    if task.amount >= task.stackSize then
        -- Simulates right-clicking the item, sending it straight to mail
        C_Container.UseContainerItem(task.bag, task.slot)
        table.remove(attachQueue, 1)
        C_Timer.After(0.3, ProcessAttachQueue)
        return
    end

    -- SCENARIO B: We need to split the stack
    local emptyBag, emptySlot = FindEmptyBagSlot()
    if not emptyBag then
        print("|cFF00FF00[AutoMail]|r: Your bags are full! You need at least 1 empty slot to split items.")
        attachQueue = {}
        isAttaching = false
        return
    end

    -- Step 1: Split the exact amount needed
    C_Container.SplitContainerItem(task.bag, task.slot, task.amount)
    
    local cursorRetries = 0
    local function Step2_WaitForCursor()
        if CursorHasItem() then
            -- Step 3: Drop the split stack into the empty bag slot
            C_Container.PickupContainerItem(emptyBag, emptySlot)
            
            local bagRetries = 0
            local function Step4_WaitForBag()
                local newInfo = C_Container.GetContainerItemInfo(emptyBag, emptySlot)
                -- Verify the item actually arrived in the bag
                if newInfo and newInfo.itemID == task.itemID then
                    -- Step 5: Right-click the new split stack into the mail
                    C_Container.UseContainerItem(emptyBag, emptySlot)
                    table.remove(attachQueue, 1)
                    
                    -- Move to the next task after a brief pause
                    C_Timer.After(0.3, ProcessAttachQueue)
                else
                    bagRetries = bagRetries + 1
                    if bagRetries < 20 then
                        C_Timer.After(0.1, Step4_WaitForBag)
                    else
                        print("|cFF00FF00[AutoMail]|r: Timed out waiting for item to appear in bag.")
                        attachQueue = {}
                        isAttaching = false
                    end
                end
            end
            C_Timer.After(0.1, Step4_WaitForBag)
            
        else
            cursorRetries = cursorRetries + 1
            if cursorRetries < 20 then
                C_Timer.After(0.1, Step2_WaitForCursor)
            else
                print("|cFF00FF00[AutoMail]|r: Split timed out waiting for cursor.")
                ClearCursor()
                attachQueue = {}
                isAttaching = false
            end
        end
    end
    
    -- Start polling for the cursor
    C_Timer.After(0.1, Step2_WaitForCursor)
end

-------------------------------------------------
-- Main Attaching Logic
-------------------------------------------------
local function StartAttachingItems()
    if isAttaching then return end
    if not AutoMailDB or not AutoMailDB.itemList then return end

    local requiredItems = {}
    local itemCountLimit = 0
    local hitLimit = false

    -- 1. Parse the text box string into a table (Max 12 items)
    for itemID, qty in string.gmatch(AutoMailDB.itemList, "(%d+)%s+(%d+)") do
        if itemCountLimit >= 12 then
            hitLimit = true
            break
        end
        requiredItems[tonumber(itemID)] = tonumber(qty)
        itemCountLimit = itemCountLimit + 1
    end

    if hitLimit then
        print("|cFF00FF00[AutoMail]|r: List contains more than 12 items. Only processing the first 12!")
    end

    if itemCountLimit == 0 then
        print("|cFF00FF00[AutoMail]|r: No valid items found in the list. Please use 'itemID amount'.")
        return
    end

    -- 2. Scan bags and calculate exact amounts needed from each stack
    attachQueue = {}
    for bag = 0, 4 do
        for slot = 1, C_Container.GetContainerNumSlots(bag) do
            local itemInfo = C_Container.GetContainerItemInfo(bag, slot)
            if itemInfo and requiredItems[itemInfo.itemID] then
                local needed = requiredItems[itemInfo.itemID]
                if needed > 0 then
                    local currentStackSize = itemInfo.stackCount or 1
                    
                    if currentStackSize > 0 then
                        local amountToTake = math.min(needed, currentStackSize)
                        
                        table.insert(attachQueue, {
                            bag = bag, 
                            slot = slot, 
                            itemID = itemInfo.itemID, -- Ensure itemID is passed for the verification step
                            amount = amountToTake,
                            stackSize = currentStackSize
                        })
                        
                        requiredItems[itemInfo.itemID] = needed - amountToTake
                    end
                end
            end
        end
    end

    -- 3. Start the queue process
    if #attachQueue > 0 then
        isAttaching = true
        ProcessAttachQueue()
    else
        print("|cFF00FF00[AutoMail]|r: No matching items from your list were found in your bags.")
    end
end

-------------------------------------------------
-- UI Setup & Event Handling
-------------------------------------------------
eventFrame:SetScript("OnEvent", function(self, event, arg1)
    if event == "ADDON_LOADED" and arg1 == addonName then
        if not AutoMailDB then
            AutoMailDB = { itemList = "" }
        end
        
    elseif event == "MAIL_SHOW" then
        if not configFrame then
            -- Create the Config Button
            local listBtn = CreateFrame("Button", nil, SendMailFrame, "UIPanelButtonTemplate")
            listBtn:SetSize(100, 22)
            listBtn:SetPoint("BOTTOMLEFT", SendMailFrame, "TOPLEFT", 60, 2)
            listBtn:SetText("Items List")
            
            -- Create the Attach Button
            local attachBtn = CreateFrame("Button", nil, SendMailFrame, "UIPanelButtonTemplate")
            attachBtn:SetSize(100, 22)
            attachBtn:SetPoint("LEFT", listBtn, "RIGHT", 5, 0)
            attachBtn:SetText("Attach Items")
            attachBtn:SetScript("OnClick", StartAttachingItems)

            -- Create the Multi-line Window
            configFrame = CreateFrame("Frame", nil, SendMailFrame, "BackdropTemplate")
            configFrame:SetSize(250, 300)
            configFrame:SetPoint("TOPLEFT", SendMailFrame, "TOPRIGHT", 10, 0)
            configFrame:Hide()
            
            configFrame.backdropInfo = {
                bgFile = "Interface\\ChatFrame\\ChatFrameBackground",
                edgeFile = "Interface\\Tooltips\\UI-Tooltip-Border",
                tile = true, tileSize = 16, edgeSize = 16,
                insets = { left = 3, right = 3, top = 3, bottom = 3 }
            }
            configFrame:ApplyBackdrop()
            configFrame:SetBackdropColor(0, 0, 0, 0.8)

            local title = configFrame:CreateFontString(nil, "OVERLAY", "GameFontNormal")
            title:SetPoint("TOP", 0, -10)
            title:SetText("Format: itemID amount (Max 12)")

            -- ScrollFrame for the Multi-line EditBox
            local scrollFrame = CreateFrame("ScrollFrame", nil, configFrame, "UIPanelScrollFrameTemplate")
            scrollFrame:SetPoint("TOPLEFT", 10, -30)
            scrollFrame:SetPoint("BOTTOMRIGHT", -30, 10)

            local editBox = CreateFrame("EditBox", nil, scrollFrame)
            editBox:SetMultiLine(true)
            editBox:SetFontObject("ChatFontNormal")
            editBox:SetWidth(190)
            editBox:SetHeight(250) 
            editBox:SetAutoFocus(false)
            editBox:EnableMouse(true)
            scrollFrame:SetScrollChild(editBox)

            -- Behavior scripts for the EditBox
            editBox:SetScript("OnEscapePressed", function(self)
                self:ClearFocus()
            end)

            scrollFrame:SetScript("OnMouseDown", function()
                editBox:SetFocus()
            end)

            -- Load saved text when opened, save text when changed
            editBox:SetText(AutoMailDB.itemList or "")
            editBox:SetScript("OnTextChanged", function(self)
                AutoMailDB.itemList = self:GetText()
            end)

            -- Toggle the window visibility
            listBtn:SetScript("OnClick", function()
                if configFrame:IsShown() then
                    configFrame:Hide()
                else
                    configFrame:Show()
                end
            end)
        end
    end
end)
```
