---
title: "Update itemInsights"
description: "Update properties of itemInsightsSettings object"
author: "simonhult"
localization_priority: Normal
ms.prod: "insights"
doc_type: "apiPageType"
---

# Update itemInsights

Namespace: microsoft.graph

[!INCLUDE [beta-disclaimer](../../includes/beta-disclaimer.md)]

Update [itemInsightsSettings](../resources/iteminsightssettings.md) object.

To learn how to get the properties of the [itemInsightsSettings](../resources/iteminsightssettings.md) object, see [get-iteminsights](organizationsettings-get-iteminsights.md).

To learn how to customize item insights privacy for your organization, see [customize insights privacy](/concepts/customize-item-insights-privacy.md). 

## Permissions

One of the following permissions is required to call this API. To learn more, including how to choose permissions, see [Permissions](/graph/permissions-reference).

|Permission type      | Permissions (from least to most privileged)              |
|:--------------------|:---------------------------------------------------------|
|Delegated (work or school account) | Organization.ReadWrite.All |
|Delegated (personal Microsoft account) | Not supported.    |
|Application | Organization.ReadWrite.All |

>**Note:** Using delegated permissions for this operation requires the signed-in user to have a global administrator role.

## HTTP request
<!-- { "blockType": "ignored" } -->

```http
PATCH /organization/settings/itemInsights
```

## Request headers

| Header       | Value|
|:-----------|:------|
| Authorization  | Bearer {token}. Required.  |
| Content-Type  | application/json  |

## Request body

In the request body, supply the values for relevant fields that should be updated. Existing properties that are not included in the request body will maintain their previous values or be recalculated based on changes to other property values. For best performance you shouldn't include existing values that haven't changed.

| Property	   | Type	|Description|
|:---------------|:--------|:----------|
|isEnabledInOrganization|Boolean|(default true) When set to 'true', all users' item insights are enabled, except for members of one Azure Active Directory group, if it specified by the 'disabledForGroup' property. When set to 'false', all users' item insights are disabled without any exceptions.|
|disabledForGroup|String|(default empty) an ID of Azure AD group, whose members' item insight are disabled|

## Response

If successful, this method returns a `200 OK` response code and [itemInsightsSettings](../resources/iteminsightssettings.md) object in the response body.

>**Note:** This endpoint verifies validity of a property value but does not check existence of an Azure Active Directory Group. This means, if you configured a Azure AD group that did not exist or was deleted after, then this method will show previously predefined value of 'disabledForGroup' property but item insights of these group members might be enabled. 

## Example 

##### Request

Here is an example request on how admin updates 'disabledForGroup' privacy setting in order to prohibit displaying users' item insights of an particular Azure AD group.
<!-- {
  "blockType": "request",
  "name": "update_iteminsightssettings"
}-->

```http
PATCH https://graph.microsoft.com/beta/organization/settings/itemInsights
Content-type: application/json

{
  "disabledForGroup": "edbfe4fb-ec70-4300-928f-dbb2ae86c981"
}
```

##### Response

Here is an example of the response. Note: The response object shown here may be truncated for brevity. All of the properties will be returned from an actual call.
<!-- {
  "blockType": "response",
  "truncated": true,
  "@odata.type": "microsoft.graph.itemInsightsSettings",
  "name": "update_iteminsightssettings"
} -->

```http
HTTP/1.1 200 OK
Content-type: application/json

{
  "isEnabledInOrganization": true,
  "disabledForGroup": "edbfe4fb-ec70-4300-928f-dbb2ae86c981"
}
```