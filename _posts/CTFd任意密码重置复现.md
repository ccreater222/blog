date: 2020-04-01
categories:
- cve复现
title: CTFd任意密码重置复现
---
# CTFd任意密码重置复现

今天刚好做到一题CTFd的就顺便复现下

这是补丁的信息:`Strip spaces on registration and refine password reset`

根据补丁信息去找register的代码

```python
def register():
    errors = get_errors()
    if request.method == "POST":
        name = request.form["name"]
        email_address = request.form["email"]
        password = request.form["password"]

        name_len = len(name) == 0
        names = Users.query.add_columns("name", "id").filter_by(name=name).first()
        emails = (
            Users.query.add_columns("email", "id")
            .filter_by(email=email_address)
            .first()
        )
        pass_short = len(password.strip()) == 0
        pass_long = len(password) > 128
        valid_email = validators.validate_email(request.form["email"])
        team_name_email_check = validators.validate_email(name)

        if not valid_email:
            errors.append("Please enter a valid email address")
        if email.check_email_is_whitelisted(email_address) is False:
            errors.append(
                "Only email addresses under {domains} may register".format(
                    domains=get_config("domain_whitelist")
                )
            )
        if names:
            errors.append("That user name is already taken")
        if team_name_email_check is True:
            errors.append("Your user name cannot be an email address")
        if emails:
            errors.append("That email has already been used")
        if pass_short:
            errors.append("Pick a longer password")
        if pass_long:
            errors.append("Pick a shorter password")
        if name_len:
            errors.append("Pick a longer user name")

        if len(errors) > 0:
            return render_template(
                "register.html",
                errors=errors,
                name=request.form["name"],
                email=request.form["email"],
                password=request.form["password"],
            )
        else:
            with app.app_context():
                user = Users(
                    name=name.strip(),
                    email=email_address.lower(),
                    password=password.strip(),
                )
                db.session.add(user)
                db.session.commit()
                db.session.flush()

                login_user(user)

                if config.can_send_mail() and get_config(
                    "verify_emails"
                ):  # Confirming users is enabled and we can send email.
                    log(
                        "registrations",
                        format="[{date}] {ip} - {name} registered (UNCONFIRMED) with {email}",
                    )
                    email.verify_email_address(user.email)
                    db.session.close()
                    return redirect(url_for("auth.confirm"))
                else:  # Don't care about confirming users
                    if (
                        config.can_send_mail()
                    ):  # We want to notify the user that they have registered.
                        email.sendmail(
                            request.form["email"],
                            "You've successfully registered for {}".format(
                                get_config("ctf_name")
                            ),
                        )

        log("registrations", "[{date}] {ip} - {name} registered with {email}")
        db.session.close()

        if is_teams_mode():
            return redirect(url_for("teams.private"))

        return redirect(url_for("challenges.listing"))
    else:
        return render_template("register.html", errors=errors)

```

阅读代码发现,查询用户名是否重复是直接将我们的输入拿去查询

```python
		name = request.form["name"]
        email_address = request.form["email"]
        password = request.form["password"]

        name_len = len(name) == 0
        names = Users.query.add_columns("name", "id").filter_by(name=name).first()#这里这里
        emails = (
            Users.query.add_columns("email", "id")
            .filter_by(email=email_address)
            .first()
        )
        if names:
            errors.append("That user name is already taken")
```

而验证没有重复后,则将我们输入的用户名strip后,在添加到数据库,也就是说我们可以在数据库弄多个同名账号

```python
			user = Users(
                    name=name.strip(),
                    email=email_address.lower(),
                    password=password.strip(),
                )
```



CTFd重置密码会收到这样的email

```
Did you initiate a password reset? Click the following link to reset your password:

http://f04e02df-7759-45ba-a2b2-b89399e91ca0.node3.buuoj.cn/reset_password/ImFkbWluIg.XoRn7w.CDy8yi95cLlHsZN_YRiCp9Z-mX4
```

我们去找`/reset_password/<xxx>`的路由

```python
@auth.route("/reset_password", methods=["POST", "GET"])
@auth.route("/reset_password/<data>", methods=["POST", "GET"])
@ratelimit(method="POST", limit=10, interval=60)
def reset_password(data=None):
    if data is not None:
        try:
            name = unserialize(data, max_age=1800)
        except (BadTimeSignature, SignatureExpired):
            return render_template(
                "reset_password.html", errors=["Your link has expired"]
            )
        except (BadSignature, TypeError, base64.binascii.Error):
            return render_template(
                "reset_password.html", errors=["Your reset token is invalid"]
            )
        if request.method == "GET":
            return render_template("reset_password.html", mode="set")
        if request.method == "POST":
            user = Users.query.filter_by(name=name).first_or_404()
            user.password = request.form["password"].strip()
            db.session.commit()
            log(
                "logins",
                format="[{date}] {ip} -  successful password reset for {name}",
                name=name,
            )
            db.session.close()
            return redirect(url_for("auth.login"))
```

我们发现,重置密码是根据用户名查出的第一个记录进行修改`user = Users.query.filter_by(name=name).first_or_404()`

这样我们就成功覆盖了之前真实用户的密码而不是我们创建的用户

