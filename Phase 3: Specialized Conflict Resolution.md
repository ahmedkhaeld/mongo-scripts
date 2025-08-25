## Overview
This third pipeline is a post-processing step that runs on the GraduationUsersConflicts collection. It identifies and categorizes users whose conflicts are specifically due to National ID duplication, implementing the business rule that UserProd is the source of truth for National ID conflicts.

### Pipeline Purpose

Source Collection: GraduationUsersConflicts (conflicts identified in Phase 2) <br>
Target Collection: GraduationUsersSkippedByNationalId (specialized conflict bucket) <br>
Business Logic: When National ID already exists in User(Prod), skip the graduation user <br>


```
[
  {
    $lookup: {
      from: "UserProd",
      let: { nationalId: "$personalInformation.nationalId" },
      pipeline: [
        {
          $match: {
            $expr: { $eq: ["$personalInformation.nationalId", "$$nationalId"] }
          }
        },
        {
          $project: {
            _id: 1,
            email: 1,
            "personalInformation.nationalId": 1,
            "personalInformation.phoneNumber": 1,
            "personalInformation.countryCode": 1
          }
        },
        { $limit: 1 }
      ],
      as: "nationalIdConflict"
    }
  },
  {
    $match: {
      nationalIdConflict: { $ne: [] }  // Has nationalId conflict
    }
  },
  {
    $addFields: {
      skipReason: "NationalId already exists in UserProd - UserProd is source of truth",
      graduationUserData: {
        _id: "$_id",
        email: "$email",
        nationalId: "$personalInformation.nationalId",
        phoneNumber: "$personalInformation.phoneNumber",
        countryCode: "$personalInformation.countryCode",
        firstName: "$personalInformation.firstName",
        lastName: "$personalInformation.lastName"
      },
      existingUserData: { $arrayElemAt: ["$nationalIdConflict", 0] },
      skippedAt: new Date()
    }
  },
  {
    $project: {
      skipReason: 1,
      graduationUserData: 1,
      existingUserData: 1,
      skippedAt: 1
    }
  },
  {
    $merge: {
      into: "GraduationUsersSkippedByNationalId",
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
]

```
