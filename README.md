# e2e testing framework comparison in 2024
The aim of the project to compare the available free end to end testing frameworks in 2024.
Currently Cypress, Playwright, Selenium and Nightwatch are planned, but I might add more later.
We use a [Laravel example CRUD API](https://github.com/blablabla1234678/proba) to test against.

## Laravel
Laravel has [PhpUnit based feature tests](https://github.com/blablabla1234678/proba/blob/main/tests/Feature/PostTest.php#L87C1-L97C6), we can use as baseline.
Laravel supports only a single request per test, so we have to use the database directly to add fixtures and check the results.
As of the e2e frameworks the tests depend on each other, even the test fiiles depend on each other, which is not the best practice.
I use helper functions to get the id of certain users, their token and the id of certain posts to reuse some code.

```php
public function test_deleting_post():void {
	$container = new Container();
	$data = new Data();
	$container->createUser($data->user1);
	$post = $container->createPost($data->post1);
	$this->assertEquals($container->countPosts(), 1);
	$this->withToken($container->createToken())
		->deleteJson('/api/posts/'.$post->id)
		->assertStatus(200);
	$this->assertEquals($container->countPosts(), 0);
}
```

## Cypress

I think the main problem with Cypress that we have to fight against callback hell. With aliases we can try to workaround it, but still it is not the best experience.
Another issue that I did not find matchers for JSON comparison for the scenario where the first JSON contains more properties than the second and we need to ignore those properties.
These two problems make the test files extremely long though I still managed to [write something readable](https://github.com/blablabla1234678/proba-cypress/blob/main/cypress/e2e/posts.spec.cy.js#L52).

```js
it('can read and delete posts', () => {
	cy.getUserWithToken('user1b');
	cy.getPost('post1b');

	cy.get('@post1b')
		.then((post) => cy.request({
			method: 'GET', 
			url: `posts/${post.id}`
		}))
		.then((response) => {
			expect(response.status).to.eq(200);
			expect(response.body.title).to.eq(data.post1b.title);
			expect(response.body.body).to.eq(data.post1b.body);
		});

	cy.aliases(['user1b', 'post1b'])
		.then(([user, post]) => cy.request({
			method: 'DELETE',
			url: `posts/${post.id}`,
			headers: {
				authorization: 'Bearer '+user.token.plainText
			}
		}))
		.then((response) => {
			expect(response.status).to.eq(200);
		});

	cy.get('@post1b')
		.then((post) => cy.request({
			method: 'GET', 
			url: `posts/${post.id}`,
			failOnStatusCode: false
		}))
		.then((response) => {
			expect(response.status).to.eq(404);
		});
});
```

## Playwright
Playwright [is like the fresh air](https://github.com/blablabla1234678/proba-playwright/blob/main/tests/02-posts.spec.js#L46) after Cypress. It supports async/await extensively. My only problem that I always have to call `await response.json()` instead of just getting `response.body`.
Another little issue that the `body` is called `data` in the requests, which is a weird naming if you ask me, but it is acceptable.
It supports partial JSON comparison as well, so I can solve it with a one-liner instead of two or more lines.

```js
test('can read and delete posts', async ({request}) => {
	const user = await getUserWithToken(request, 'user1');
	const post = await getPost(request, 'post1b');

	let response = await request.get(`posts/${post.id}`);
	expect(response.status()).toBe(200);
	let body = await response.json();
	expect(body).toEqual(expect.objectContaining(postsFixture.post1b));

	response = await request.delete(`posts/${post.id}`, {
		headers: {
			authorization: 'Bearer '+user.token.plainText
		}
	});
	expect(response.status()).toBe(200);

	response = await request.get(`posts/${post.id}`);
	expect(response.status()).toBe(404);
});
```

