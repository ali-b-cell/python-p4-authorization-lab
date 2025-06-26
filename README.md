# Authorizing Requests Lab

## Learning Goals

- Use the session object to authorize a user to perform actions in an app.

***

## Key Vocab

- **Identity and Access Management (IAM)**: a subfield of software engineering that
  focuses on users, their attributes, their login information, and the resources
  that they are allowed to access.
- **Authentication**: proving one's identity to an application in order to
  access protected information; logging in.
- **Authorization**: allowing or disallowing access to resources based on a
  user's attributes.
- **Session**: the time between a user logging in and logging out of a web
  application.
- **Cookie**: data from a web application that is stored by the browser. The
  application can retrieve this data during subsequent sessions.

***

## Introduction

In this lab, we'll continue working on the blog site, and add some features that
only logged in users have access to.

There is some starter code in place for a Flask API backend and a React frontend.
To get set up, run:

```console
$ pipenv install; pipenv shell
$ npm install --prefix client
$ cd server
$ flask db upgrade
$ python seed.py
```

You can work on this lab by running the tests with `pytest -x`. It will also be
helpful to see what's happening during the request/response cycle by running the
app in the browser. You can run the Flask server with:

```console
$ python app.py
```

And you can run React in another terminal from the root project directory with:

```console
$ npm start --prefix client
```

You don't have to make any changes to the React code to get this lab working.

***

## Instructions

Now that we've got the basic login feature working, let's reward our logged
in users with some bonus content that only users who have logged in will be able
to access!

We added a new attribute to our articles, `is_member_only`, to reflect whether
the article should only be available to authorized users of the site. We also
created two new views: `MemberOnlyIndex` and `MemberOnlyArticle`.

Your goal is to add the following functionality to the new views:

- If a user is not signed in, the `get()` methods in each view should return a
  status code of 401 unauthorized, along with an error message.
- If the user is signed in, the `get()` methods in each view should return the
  JSON data for the members-only articles and the members-only article by ID, respectively.

***

## Resources

- [What is Authentication? - auth0](https://auth0.com/intro-to-iam/what-is-authentication)
- [API - Flask: class flask.session](https://flask.palletsprojects.com/en/2.2.x/api/#flask.session)







from flask import Flask, make_response, jsonify, request, session
from flask_migrate import Migrate
from flask_restful import Api, Resource

from models import db, Article, User

app = Flask(__name__)
app.secret_key = b'Y\xf1Xz\x00\xad|eQ\x80t \xca\x1a\x10K'
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///app.db'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.json.compact = False

migrate = Migrate(app, db)
db.init_app(app)
api = Api(app)

# ---------- ROUTES ----------

class ClearSession(Resource):
    def delete(self):
        session['page_views'] = None
        session['user_id'] = None
        return {}, 204

class IndexArticle(Resource):
    def get(self):
        articles = [article.to_dict() for article in Article.query.all()]
        return make_response(jsonify(articles), 200)

class ShowArticle(Resource):
    def get(self, id):
        article = Article.query.filter(Article.id == id).first()
        article_json = article.to_dict()

        if not session.get('user_id'):
            session['page_views'] = 0 if not session.get('page_views') else session['page_views']
            session['page_views'] += 1

            if session['page_views'] <= 3:
                return article_json, 200
            return {'message': 'Maximum pageview limit reached'}, 401

        return article_json, 200

class Login(Resource):
    def post(self):
        username = request.get_json().get('username')
        user = User.query.filter(User.username == username).first()

        if user:
            session['user_id'] = user.id
            return user.to_dict(), 200

        return {}, 401

class Logout(Resource):
    def delete(self):
        session['user_id'] = None
        return {}, 204

class CheckSession(Resource):
    def get(self):
        user_id = session.get('user_id')
        if user_id:
            user = User.query.filter(User.id == user_id).first()
            return user.to_dict(), 200
        return {}, 401

class MemberOnlyIndex(Resource):
    def get(self):
        if not session.get('user_id'):
            return {'error': 'Unauthorized'}, 401

        articles = Article.query.filter_by(is_member_only=True).all()
        return [article.to_dict() for article in articles], 200

class MemberOnlyArticle(Resource):
    def get(self, id):
        if not session.get('user_id'):
            return {'error': 'Unauthorized'}, 401

        article = Article.query.filter_by(id=id, is_member_only=True).first()
        if not article:
            return {'error': 'Article not found'}, 404

        return article.to_dict(), 200

# ---------- REGISTER RESOURCES ----------
api.add_resource(ClearSession, '/clear', endpoint='clear')
api.add_resource(IndexArticle, '/articles', endpoint='article_list')
api.add_resource(ShowArticle, '/articles/<int:id>', endpoint='show_article')
api.add_resource(Login, '/login', endpoint='login')
api.add_resource(Logout, '/logout', endpoint='logout')
api.add_resource(CheckSession, '/check_session', endpoint='check_session')
api.add_resource(MemberOnlyIndex, '/members_only_articles', endpoint='member_index')
api.add_resource(MemberOnlyArticle, '/members_only_articles/<int:id>', endpoint='member_article')

# ---------- RUN ----------
if __name__ == '__main__':
    app.run(port=5555, debug=True)
