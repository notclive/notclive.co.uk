---
title: Investigating an Increase in Response Times After Upgrading to Spring Boot 2.1
---

### {{ page.date | date: "%B %Y" }}

# {{page.title}}

## Problem

Before a recent release our performance tests caught an increase in response times. Specifically we saw a roughly 100ms increase in response times across all endpoints in one of our services. I looked through the changes to that service since the last release, a Spring Boot upgrade from version 2.0 to 2.1 stood out. Reverting the upgrade proved that it was the casue, but I didn't know why.

## Cause

I tracked the additional 100ms down to the BCryptPasswordEncoder.matches method. This method compares a provided password to a BCrypt encoded version of the password.

From Spring Security 5 passwords are encoded by default, encoding passwords serves two purposes.
1. Password comparison time doesn't vary based on password size, this prevents timing attacks
2. Password comparison is slower, this prevents brute force attacks

Slowing down password comparisons to 100ms would be acceptable and sensible for a typical application where the client logs in once per session, and then presents a session token for subsequent requests. Our service doesn't operate in this way, there is no session state on the server side or in the client, clients sends their credentials on each request.

## Solution

We needed an encoder that behaved similarly to the default Spring encoder but was faster. 
Here's how I configured Spring to use a BCryptPasswordEncoder with a strength of 4, rather than default strength of 10, at this strength comparisons only take ~3ms.

```
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    private final UserDetailsService userDetailsService;

    @Autowired
    public WebSecurityConfig(UserDetailsService userDetailsService) {
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(userDetailsService).passwordEncoder(createFasterThanDefaultPasswordEncoder());
    }

    /**
    * The default passwordEncoder defined in {@link org.springframework.security.crypto.factory.PasswordEncoderFactories}
    * uses a very strong but slow bcrypt encoder that takes ~100ms to check a password.
    * We use a faster encoder because an overhead of 100ms per request isn't acceptable.
    * The deprecated no-op encoder is required to decode the un-encoded password on the first request,
    * the encoding will then be upgraded to bcrypt by {@link org.springframework.security.authentication.dao.DaoAuthenticationProvider}.
    */
    @SuppressWarnings("deprecation")
    public PasswordEncoder createFasterThanDefaultPasswordEncoder() {
        return new DelegatingPasswordEncoder("bcrypt", ImmutableMap.of(
                "bcrypt", new BCryptPasswordEncoder(4),
                "noop", org.springframework.security.crypto.password.NoOpPasswordEncoder.getInstance()
        ));
    }

    ...

```