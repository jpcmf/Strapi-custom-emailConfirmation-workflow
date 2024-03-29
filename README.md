# Strapi-custom-emailConfirmation-workflow

# Answering a [question on Strapi Forums](https://forum.strapi.io/t/extending-api-auth-email-confirmation-confirmation-xxx-endpoint-with-custom-logic/36969)


Overwrite the email confirmation method with almost a copy-paste to be able to call your own `emailConfirmation` method that can be (again) almost a copy-paste but adding the user's email to the email confirmation redirect URL.

---

For future reference for someone else, I was able to overwrite the `emailConfirmation` function and attach the user's email to the URL. This allows me to handle it in my front-end. Follow these steps:

1. Create a file named `strapi-server.js` in the directory `src/extensions/users-permissions/`
2. Add the following code:

```js
const _ = require("lodash");
const utils = require("@strapi/utils");
const { getService } = require("@strapi/plugin-users-permissions/server/utils");
const {
  validateEmailConfirmationBody
} = require("@strapi/plugin-users-permissions/server/controllers/validation/auth");

const { ValidationError } = utils.errors;
const { sanitize } = utils;
const sanitizeUser = (user, ctx) => {
  const { auth } = ctx.state;
  const userSchema = strapi.getModel("plugin::users-permissions.user");

  return sanitize.contentAPI.output(user, userSchema, { auth });
};

module.exports = plugin => {
  plugin.controllers.auth.emailConfirmation = async (ctx, next, returnUser) => {
    const { confirmation: confirmationToken } = await validateEmailConfirmationBody(ctx.query);

    const userService = getService("user");
    const jwtService = getService("jwt");

    const [user] = await userService.fetchAll({
      filters: { confirmationToken }
    });

    if (!user) {
      throw new ValidationError("Invalid token");
    }

    await userService.edit(user.id, {
      confirmed: true,
      confirmationToken: null
    });

    if (returnUser) {
      ctx.send({
        jwt: jwtService.issue({ id: user.id }),
        user: await sanitizeUser(user, ctx)
      });
    } else {
      const settings = await strapi.store({ type: "plugin", name: "users-permissions", key: "advanced" }).get();

      ctx.redirect(settings.email_confirmation_redirection + "?email=" + user.email || "/");
    }
  };

  return plugin;
};
```
