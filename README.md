# Stroy

Imagine you are building a simple project management application that has follwing simple features 

- Manage projects (your organisation may have multiple projects running)
- Manage work packages related to a project
- Manage members in your organisation that may need to have access to projects in your organisation.

# Shared database approach 

Now in this writing we will see how a single shared database can hold information and enforce multi tenant constraints in our application. 
For the implementation we will consider following collections

- Users
- Organisations
- Memberships
- Projects
- WorkPackages

As we are more focused to learn about multi tenancy, we will keep the database schema short, simple and relavant to our needs.

# The database schema

These are the simplified schema for our application

1. **users**:
```javascript
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  name: String,
  email: String,
  // other relevant fields
});

const User = mongoose.model('User', userSchema);

module.exports = User;
```

2. **organisations**:
```javascript
const mongoose = require('mongoose');

const organisationSchema = new mongoose.Schema({
  name: String,
  // other relevant fields
});

const Organisation = mongoose.model('Organisation', organisationSchema);

module.exports = Organisation;
```

3. **memberships**:
```javascript
const mongoose = require('mongoose');

const membershipSchema = new mongoose.Schema({
  user: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  },
  organisation: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Organisation'
  },
  role: {
    type: String,
    default:"Admin",
    enum: ["Admin", "Member", "Viewer"]
  }
  // other relevant fields
});

const Membership = mongoose.model('Membership', membershipSchema);

module.exports = Membership;
```

4. **projects**:
```javascript
const mongoose = require('mongoose');

const projectSchema = new mongoose.Schema({
  name: String,
  // other relevant fields
});

const Project = mongoose.model('Project', projectSchema);

module.exports = Project;
```

5. **work-packages**:
```javascript
const mongoose = require('mongoose');

const workPackageSchema = new mongoose.Schema({
  name: String,
  project: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Project'
  },
  // other relevant fields
});

const WorkPackage = mongoose.model('WorkPackage', workPackageSchema);

module.exports = WorkPackage;
```

# Attention

Remember the feature requirements? Notice in our current schema there is no way we can tell the relationships between the projects 
and organisations or between the work-packages and the organisations. This means there is no tenancy feature yet.

Also notice I have a membership collection that defines the relationship between a user and an organisation with appropriate user role.
These information will be usefull later.

# Further development 

By now you might be thinking to estabilish the relations, we can create some relationship collections just like **memberships** or we can
simply put the `organisation` attribute in the **projects** and **work-packages** collections. 

I would choose the later (the rationale behind this decision is a topic itself to be discussed) and put the following codes in the **projects** and **work-packages** collections. 
```javascript
organisation: {
  type: mongoose.Schema.Types.ObjectId,
  ref: 'Organisation'
},
```
So now the schemas look like this

1. **projects**:
```javascript
const mongoose = require('mongoose');

const projectSchema = new mongoose.Schema({
  name: String,
  organisation: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Organisation'
  },
  // other relevant fields
});

const Project = mongoose.model('Project', projectSchema);

module.exports = Project;
```
2. **work-packages**:
```javascript
const mongoose = require('mongoose');

const workPackageSchema = new mongoose.Schema({
  name: String,
  project: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Project'
  },
  organisation: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Organisation'
  },
  // other relevant fields
});

const WorkPackage = mongoose.model('WorkPackage', workPackageSchema);

module.exports = WorkPackage;
```

# But hold on

Let's look at few things from a tenant query perspective. We don't want to be accessing wrong `organisation/tenants` data even by 
mistake. Hence We need to scope our queries with `organisation` specific filter to prevent data leakage. A simple implmentation is as follows:

```javascript
const Project = require('./models/Project');

// Find all projects for an organisation
Project.find({ organisation: 'organisation id that is found from token', ....other queries here.... });
```

```javascript
const WorkPackage = require('./models/WorkPackage');

// Find all work packages for an organisation
WorkPackage.find({ organisation: 'organisation id that is found from token' , ....other queries here.... });
```

> Imagine how hectic will it be to deal with this implementation. You need to give proper attention to places where scope filter is needed. If a 
developer makes a mistake and forgets to add the `organisation` filter in places where it's needed, that's a big reputational damage. 

# Rather build a plugin

> A real life saver for the problem would be a method that acts like `find` but also does the necessary checks for you and enforces you to apply `organisation/tenant` scoped filter, wouldn't it? Let's call it `findByOrg`.

We know `mongoose` comes up with a lot of handy features and one of them is the ability to add a plugin to a schema. Instead of copying
and pasting the following codes in every collection,

```javascript
organisation: {
  type: mongoose.Schema.Types.ObjectId,
  ref: 'Organisation'
},
```

we will develop a plugin to inject this property in the collections that need tenant scoping and we will also add some helpful functions
for better query scoping.

# The plugin

Heads up: If you don't know how mongoose plugins work, read through the docs here [Plugins](https://mongoosejs.com/docs/plugins.html) before you proceed.

Follwing code is a simplified version of my code that I already have used to write an application before that is currently in production.
```javascript
const mongoose = require("mongoose");
const mongoosePaginate = require("mongoose-paginate-v2");

/**
 * this plugin allowes orgdata feature for  data-models
 * @param {import("mongoose").Schema} schema
 */
const orgDataPlugin = (schema) => {
  schema.add({
    organisation: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'Organisation'
    },
  });
  if (
    !schema.statics.paginate ||
    typeof schema.statics.paginate === "function"
  ) {
    mongoosePaginate(schema);
  }
  schema.statics._orgIdValidation = function (orgId) {
    if (!orgId) throw new Error("orgId at arg[0] is a required.");
    if (!mongoose.Types.ObjectId.isValid(orgId))
      throw new Error("invalid mongodb ObjectId");
    return true;
  };
  schema.statics.paginateByOrg = function (
    orgId,
    query,
    ...paginationRestArgs
  ) {
    if (this._orgIdValidation(orgId)) {
      return this.paginate(
        {
          ...query,
          // CAUTION: it organization has to be always at the end of query here, otherwise huge security risk
          organisation: orgId,
        },
        ...paginationRestArgs
      );
    }
  };
  schema.statics.countDocumentsByOrg = function (
    orgId,
    query,
    ...paginationRestArgs
  ) {
    if (this._orgIdValidation(orgId)) {
      return this.countDocuments(
        {
          ...query,
          // CAUTION: it organization has to be always at the end of query here, otherwise huge security risk
          organisation: orgId,
        },
        ...paginationRestArgs
      );
    }
  };
  schema.statics.findOneByOrg = function (orgId, query, ...args) {
    if (this._orgIdValidation(orgId)) {
      return this.findOne(
        {
          ...query,
          // CAUTION: it organization has to be always at the end of query here, otherwise huge security risk
          organisation: orgId,
        },
        ...args
      );
    }
  };
  schema.statics.findByOrg = function (orgId, query, ...args) {
    if (this._orgIdValidation(orgId)) {
      return this.find(
        {
          ...query,
          // CAUTION: it organization has to be always at the end of query here, otherwise huge security risk
          organisation: orgId,
        },
        ...args
      );
    }
  };
  schema.statics.findOneAndUpdateByOrg = function (orgId, query, ...args) {
    if (this._orgIdValidation(orgId)) {
      return this.findOneAndUpdate(
        {
          ...query,
          // CAUTION: it organization has to be always at the end of query here, otherwise huge security risk
          organisation: orgId,
        },
        ...args
      );
    }
  };
  schema.statics.updateManyByOrg = function (orgId, query, ...args) {
    if (this._orgIdValidation(orgId)) {
      return this.updateMany(
        {
          ...query,
          // CAUTION: it organization has to be always at the end of query here, otherwise huge security risk
          organisation: orgId,
        },
        ...args
      );
    }
  };
};
module.exports = { orgDataPlugin };
```

> Excuse some of my repeatative codes here in this block, I made them purposefully not to complicate things. 

See here we have extended the existing static methods to imrove tenant based scoping consistency. 

# Use this plugin 

To use the plugin is very simple. Just plug it into your model and you get all the functions including `organisation` reference attribute
for setting up the relationship.

```javascript
const mongoose = require('mongoose');
const { orgDataPlugin } = require("./orgDataPlugin")
const projectSchema = new mongoose.Schema({
  name: String,
  // organisation: {
  //  type: mongoose.Schema.Types.ObjectId,
  //  ref: 'Organisation'
  // },
  // other relevant fields
});

projectSchema.plugin(orgDataPlugin);
const Project = mongoose.model('Project', projectSchema);

module.exports = Project;
```

Notice here we commented the `oranisation` attribute and don't need to use it explecitely anymore because that is bound inside
the plugin. I only commented it to explain things. You can totally get rid of the commented block. Same approach is applicable for work-packages or any other collection that needs `organisation/tenant` scope

Let's see the plugin in action. Remember our scoped query ?

```javascript
const Project = require('./models/Project');

// Find all projects for an organisation
Project.find({ organisation: 'organisation id that is found from token', ....other queries here.... });

const WorkPackage = require('./models/WorkPackage');

// Find all work packages for an organisation
WorkPackage.find({ organisation: 'organisation id that is found from token', ....other queries here.... });
```

That currently looks like this 

```javascript
const Project = require('./models/Project');

// Find all projects for an organisation
Project.findByOrg('organisation id that is found from token', { ....other queries here.... });

const WorkPackage = require('./models/WorkPackage');

// Find all work packages for an organisation
WorkPackage.findByOrg('organisation id that is found from token', { ....other queries here.... } );
```

So, now you can see wherever the organisation scope is needed, we can use `findByOrg` and be sure that if we put any bad query that will not mix up data between tenants and this will prevent data breach because if you notice the plugin you will see the `organisation` is always being appended at the end. Moreover, if a valid `organisationId` is not present in the first argument this will result in an error.
So hard fixing the `organisation` with its `id` protects you from accessing wrong organisations data.

Also when reviewing codes and developing the features developers are forced to use the `findByOrg` function and this implements a consistant practice arocss the project.

# Bonus

See in this writing I have used the term `organisation id that is found from token` in few places. You may be thinking how is that handled?
Well it's really simple. I'm leaving the logics here.

```javascript
// you may call it login or signin
function signUserIdentity() {
  // verify user email existance
  
  // match user passwords
  
  // track which organisastion user is trying to login using `organisation/tenant` id
  
  // match and verfy the membership

  const tokenPayload = {
    userId:  'id from db',
    organisationId: 'organisation's id that is found from matching membership'
    role: 'role in the organisation that is found from matching membership'
  }
  
  // sign the json webtokens (accessTokens and refreshTokens)
  
  // send the token in response and https secure cookies 
}
```

In your authentication middleware just inject the decoded payload from the `accessToken`.


```javascript
// you may call it authUser
function deserialise(req,res,next) {
  // complex authentications and verificaion logics...

  const decodedDataAfterTokenAuthentication = ....logics....
  
  // decodedDataAfterTokenAuthentication looks like the following
  // {
  //    userId:  'id from db',
  //    organisationId: 'organisation's id that is found from matching membership'
  //    role: 'role in the organisation that is found from matching membership'
  // }
   
  req.accessControl = {
    ...decodedDataAfterTokenAuthentication
  }
  next()
}
```

Now in your services use it as following

```javascript
await Project.findByOrg(req.accessControl.organisationId, { ....other queries here.... });
```

That's it everyone, that is a very good starting point for you to implment a single shared db multi-tenant architecture. Let me know 
your feedacks or any other approches you think is a better fit.


## Want to have a chat?

My Email: dev.reyadhossain@gmail.com
My Linkedin: [MD. Reyad Hossain](https://www.linkedin.com/in/md-reyad-hossain-3036ab194)
Phone: +8801644417464
