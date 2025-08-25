### Overview
This document outlines the MongoDB aggregation pipeline designed to migrate graduation users from the GraduationUser collection to the production User collection with duplicate prevention.
Prerequisites

Source Data: GraduationUser collection must exist and contain user records to be migrated <br>
Target Collection: User collection (production user collection) must be accessible <br>
Migration Goal: Safely transfer graduation users to production while preventing duplicate entries<br>

### Migration Strategy
Phase 1: Clean Insert <br>
The current pipeline focuses on clean insertion - only inserting users that don't already exist in the production database. This conservative approach ensures data integrity during the initial migration.

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
              $or: [
                { $eq: ["$email", "$email"] },
                { $eq: ["$personalInformation.nationalId", "$nationalId"] },
                {
                  $and: [
                    { $eq: ["$personalInformation.phoneNumber", "$phone"] },
                    { $eq: ["$personalInformation.countryCode", "$code"] }
                  ]
                }
              ]
            }
          }
        },
        {
          $limit: 1  // Stop at first match for performance
        }
      ],
      as: "existingUsers"
    }
  },
  {
    $match: {
      existingUsers: { $eq: [] }  // Only users with no matches
    }
  },
  {
    $addFields: {
      insertedAt: new Date(),
      source: "GraduationUser_clean_insert"
    }
  },
  {
    $unset: "existingUsers"
  },
  {
    $merge: {
      into: "User",
      whenMatched: "fail",         // Fail if somehow a match occurs
      whenNotMatched: "insert"     // Insert only brand new users
    }
  }
]
```
