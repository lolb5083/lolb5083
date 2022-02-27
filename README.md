osiris inventory changer paste.  
#include "Inventory.h"
#include "InventoryConfig.h"

constexpr auto CONFIG_VERSION = 3;

json InventoryChanger::toJson() noexcept
{
    json j;

    j["Version"] = CONFIG_VERSION;

    for (auto& items = j["Items"]; const auto & item : Inventory::get()) {
        if (item.isDeleted())
            continue;

        json itemConfig;

        const auto& gameItem = item.get();
        itemConfig["Weapon ID"] = gameItem.weaponID;
        itemConfig["Item Name"] = StaticData::getWeaponName(gameItem.weaponID);

        switch (gameItem.type) {
        case StaticData::Type::Sticker: {
            const auto& staticData = StaticData::paintKits()[gameItem.dataIndex];
            itemConfig["Sticker ID"] = staticData.id;
            break;
        }
        case StaticData::Type::Glove: {
            const auto& staticData = StaticData::paintKits()[gameItem.dataIndex];
            itemConfig["Paint Kit"] = staticData.id;
            itemConfig["Paint Kit Name"] = staticData.name;

            const auto& dynamicData = Inventory::dynamicGloveData(item.getDynamicDataIndex());

            itemConfig["Wear"] = dynamicData.wear;
            itemConfig["Seed"] = dynamicData.seed;
            break;
        }
        case StaticData::Type::Skin: {
            const auto& staticData = StaticData::paintKits()[gameItem.dataIndex];
            itemConfig["Paint Kit"] = staticData.id;
            itemConfig["Paint Kit Name"] = staticData.name;

            const auto& dynamicData = Inventory::dynamicSkinData(item.getDynamicDataIndex());

            if (dynamicData.tournamentID != 0)
                itemConfig["Tournament ID"] = dynamicData.tournamentID;
            itemConfig["Wear"] = dynamicData.wear;
            itemConfig["Seed"] = dynamicData.seed;
            if (dynamicData.statTrak > -1)
                itemConfig["StatTrak"] = dynamicData.statTrak;

            if (!dynamicData.nameTag.empty())
                itemConfig["Name Tag"] = dynamicData.nameTag;

            auto& stickers = itemConfig["Stickers"];
            for (std::size_t i = 0; i < dynamicData.stickers.size(); ++i) {
                const auto& sticker = dynamicData.stickers[i];
                if (sticker.stickerID == 0)
                    continue;

                json stickerConfig;
                stickerConfig["Sticker ID"] = sticker.stickerID;
                stickerConfig["Wear"] = sticker.wear;
                stickerConfig["Slot"] = i;
                stickers.push_back(std::move(stickerConfig));
            }

            if (dynamicData.tournamentStage != TournamentStage{}) {
                itemConfig["Tournament Stage"] = dynamicData.tournamentStage;
                itemConfig["Tournament Team 1"] = dynamicData.tournamentTeam1;
                itemConfig["Tournament Team 2"] = dynamicData.tournamentTeam2;
                itemConfig["Tournament Player"] = dynamicData.proPlayer;
            }
            break;
        }
        case StaticData::Type::Music: {
            const auto& staticData = StaticData::paintKits()[gameItem.dataIndex];
            itemConfig["Music ID"] = staticData.id;
            if (const auto& dynamicData = Inventory::dynamicMusicData(item.getDynamicDataIndex()); dynamicData.statTrak > -1)
                itemConfig["StatTrak"] = dynamicData.statTrak;
            break;
        }
        case StaticData::Type::Patch: {
            const auto& staticData = StaticData::paintKits()[gameItem.dataIndex];
            itemConfig["Patch ID"] = staticData.id;
            break;
        }
        case StaticData::Type::Graffiti: {
            const auto& staticData = StaticData::paintKits()[gameItem.dataIndex];
            itemConfig["Graffiti ID"] = staticData.id;
            break;
        }
        case StaticData::Type::SealedGraffiti: {
            const auto& staticData = StaticData::paintKits()[gameItem.dataIndex];
            itemConfig["Graffiti ID"] = staticData.id;
            break;
        }
        case StaticData::Type::Agent: {
            const auto& dynamicData = Inventory::dynamicAgentData(item.getDynamicDataIndex());
            auto& stickers = itemConfig["Patches"];
            for (std::size_t i = 0; i < dynamicData.patches.size(); ++i) {
                const auto& patch = dynamicData.patches[i];
                if (patch.patchID == 0)
                    continue;

                json patchConfig;
                patchConfig["Patch ID"] = patch.patchID;
                patchConfig["Slot"] = i;
                stickers.push_back(std::move(patchConfig));
            }
            break;
        }
        case StaticData::Type::ServiceMedal: {
            if (const auto& dynamicData = Inventory::dynamicServiceMedalData(item.getDynamicDataIndex()); dynamicData.issueDateTimestamp != 0)
                itemConfig["Issue Date Timestamp"] = dynamicData.issueDateTimestamp;
            break;
        }
        case StaticData::Type::Case: {
            if (StaticData::cases()[gameItem.dataIndex].isSouvenirPackage()) {
                if (const auto& dynamicData = Inventory::dynamicSouvenirPackageData(item.getDynamicDataIndex()); dynamicData.tournamentStage != TournamentStage{}) {
                    itemConfig["Tournament Stage"] = dynamicData.tournamentStage;
                    itemConfig["Tournament Team 1"] = dynamicData.tournamentTeam1;
                    itemConfig["Tournament Team 2"] = dynamicData.tournamentTeam2;
                    itemConfig["Tournament Player"] = dynamicData.proPlayer;
                }
            }
        }
        default:
            break;
        }

        items.push_back(std::move(itemConfig));
    }

    if (const auto localInventory = memory->inventoryManager->getLocalInventory()) {
        auto& equipment = j["Equipment"];
        for (std::size_t i = 0; i < 57; ++i) {
            json slot;

            if (const auto itemCT = localInventory->getItemInLoadout(Team::CT, static_cast<int>(i))) {
                if (const auto soc = memory->getSOCData(itemCT); soc && Inventory::getItem(soc->itemID))
                    slot["CT"] = Inventory::getItemIndex(soc->itemID);
            }

            if (const auto itemTT = localInventory->getItemInLoadout(Team::TT, static_cast<int>(i))) {
                if (const auto soc = memory->getSOCData(itemTT); soc && Inventory::getItem(soc->itemID))
                    slot["TT"] = Inventory::getItemIndex(soc->itemID);
            }

            if (const auto itemNOTEAM = localInventory->getItemInLoadout(Team::None, static_cast<int>(i))) {
                if (const auto soc = memory->getSOCData(itemNOTEAM); soc && Inventory::getItem(soc->itemID))
                    slot["NOTEAM"] = Inventory::getItemIndex(soc->itemID);
            }

            if (!slot.empty()) {
                slot["Slot"] = i;
                equipment.push_back(std::move(slot));
            }
        }
    }

    return j;
}

[[nodiscard]] auto loadSkinStickersFromJson(const json& j) noexcept
{
    std::array<StickerConfig, 5> skinStickers;

    if (!j.contains("Stickers"))
        return skinStickers;

    const auto& stickers = j["Stickers"];
    if (!stickers.is_array())
        return skinStickers;

    for (const auto& sticker : stickers) {
        if (!sticker.is_object())
            continue;

        if (!sticker.contains("Sticker ID") || !sticker["Sticker ID"].is_number_integer())
            continue;

        if (!sticker.contains("Slot") || !sticker["Slot"].is_number_integer())
            continue;

        const int stickerID = sticker["Sticker ID"];
        if (stickerID == 0)
            continue;

        const std::size_t slot = sticker["Slot"];
        if (slot >= skinStickers.size())
            continue;

        skinStickers[slot].stickerID = stickerID;

        if (sticker.contains("Wear") && sticker["Wear"].is_number_float())
            skinStickers[slot].wear = sticker["Wear"];
    }

    return skinStickers;
}

[[nodiscard]] std::size_t loadDynamicSkinDataFromJson(const json& j) noexcept
{
    DynamicSkinData dynamicData;

    if (j.contains("Tournament ID")) {
        if (const auto& tournamentID = j["Tournament ID"]; tournamentID.is_number_unsigned())
            dynamicData.tournamentID = tournamentID;
    }

    if (j.contains("Wear")) {
        if (const auto& wear = j["Wear"]; wear.is_number_float())
            dynamicData.wear = wear;
    }

    if (j.contains("Seed")) {
        if (const auto& seed = j["Seed"]; seed.is_number_integer())
            dynamicData.seed = seed;
    }

    if (j.contains("StatTrak")) {
        if (const auto& statTrak = j["StatTrak"]; statTrak.is_number_integer())
            dynamicData.statTrak = statTrak;
    }

    if (j.contains("Name Tag")) {
        if (const auto& nameTag = j["Name Tag"]; nameTag.is_string())
            dynamicData.nameTag = nameTag;
    }

    if (j.contains("Tournament Stage")) {
        if (const auto& tournamentStage = j["Tournament Stage"]; tournamentStage.is_number_unsigned())
            dynamicData.tournamentStage = tournamentStage;
    }

    if (j.contains("Tournament Team 1")) {
        if (const auto& tournamentTeam1 = j["Tournament Team 1"]; tournamentTeam1.is_number_unsigned())
            dynamicData.tournamentTeam1 = tournamentTeam1;
    }

    if (j.contains("Tournament Team 2")) {
        if (const auto& tournamentTeam2 = j["Tournament Team 2"]; tournamentTeam2.is_number_unsigned())
            dynamicData.tournamentTeam2 = tournamentTeam2;
    }

    if (j.contains("Tournament Player")) {
        if (const auto& tournamentPlayer = j["Tournament Player"]; tournamentPlayer.is_number_unsigned())
            dynamicData.proPlayer = tournamentPlayer;
    }

    dynamicData.stickers = loadSkinStickersFromJson(j);
    return Inventory::emplaceDynamicData(std::move(dynamicData));
}

[[nodiscard]] std::size_t loadDynamicGloveDataFromJson(const json& j) noexcept
{
    DynamicGloveData dynamicData;

    if (j.contains("Wear")) {
        if (const auto& wear = j["Wear"]; wear.is_number_float())
            dynamicData.wear = wear;
    }

    if (j.contains("Seed")) {
        if (const auto& seed = j["Seed"]; seed.is_number_integer())
            dynamicData.seed = seed;
    }

    return Inventory::emplaceDynamicData(std::move(dynamicData));
}

[[nodiscard]] std::size_t loadDynamicMusicDataFromJson(const json& j) noexcept
{
    DynamicMusicData dynamicData;

    if (j.contains("StatTrak")) {
        if (const auto& statTrak = j["StatTrak"]; statTrak.is_number_integer() && statTrak > -1)
            dynamicData.statTrak = statTrak;
    }
