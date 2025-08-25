### Overview
After the clean insert pipeline, this second pipeline identifies and preserves users who were rejected due to conflicts during the initial migration. Instead of losing these records, they're stored in a dedicated collection for review and resolution.

#### Pipeline Purpose
This pipeline catches the "other side" of the migration - users from GraduationUser who were NOT inserted into User because they had conflicts with existing users.


```
[
  {
    $lookup: {
      from: "User",
      let: {
        email: "$email",
        nationalId: "$personalInformation.nationalId",
        phone: "$personalInformation.phoneNumber",
        code: "$personalInformation.countryCode"
      },
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$source", "GraduationUser_clean_insert"] }, // Only our inserted users
                {
                  $or: [
                    { $eq: ["$email", "$$email"] },  // Fixed: use $$ for let variables
                    { $eq: ["$personalInformation.nationalId", "$$nationalId"] },
                    {
                      $and: [
                        { $eq: ["$personalInformation.phoneNumber", "$$phone"] },
                        { $eq: ["$personalInformation.countryCode", "$$code"] }
                      ]
                    }
                  ]
                }
              ]
            }
          }
        }
      ],
      as: "insertedInUserProd"
    }
  },
  {
    $match: {
      insertedInUserProd: { $eq: [] } // Users NOT found in User (not inserted = have conflicts)
    }
  },
  {
    $unset: "insertedInUserProd"
  },
  {
    $addFields: {
      preservedAt: new Date(),
      status: "has_conflicts_needs_review"
    }
  },
  {
    $merge: {
      into: "GraduationUsersConflicts", // New collection
      whenMatched: "replace",
      whenNotMatched: "insert"
    }
  }
]
```

#### Next Steps 

Conflict Analysis: Review records in GraduationUsersConflicts to understand conflict patterns
Resolution Strategy: Define business rules for handling each type of conflict
