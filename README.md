# Overview

Bulwark is a fast, modern, open-source web application security engine that makes it easy to implement resilient and observable security operations for your web services.

Get started quickly with our guides and tutorials!

## This is a beta test.

Bulwark is in an early public beta test state and is not ready for production usage yet. You’re welcome to experiment with it, or check out the project [roadmap](https://docs.bulwark.security/contributing/roadmap)!

## What is Bulwark?

Bulwark is a user-friendly security engine that allows security teams to automate threat detection with a detection-as-code architecture pattern. Security teams can quickly compose powerful detection rules from reusable building-blocks while unburdening product application logic from the increased complexity of domain-specific controls.

## Detection-as-code

With Bulwark, every detection is written in a general-purpose programming language and executed within a secure sandbox. Detections are expressive and can be customized to meet domain-specific needs. They can be tested, verified, and then easily combined to form comprehensive detection suites.

Detections can also be checked into source-control and versioned like any other codebase. This makes reviewing changes straightforward and helps organizations meet audit or compliance obligations.

## Security detections

Bulwark is designed to address a wide range of security challenges. Detections may target:

* Unwanted scans
* Exploits
* Credential stuffing
* Password spraying
* Brute forcing
* Session hijacking

...and many other security use-cases. Bulwark's API enables a wide range of capabilities while giving plugin authors all of the tools needed to ensure decision results are accurate.

## Anti-fraud detections

In addition to security, you can use Bulwark to host anti-fraud functionality. Bulwark's API enables detections to use information that is typically only accessible to application logic.

Bulwark plugins can read encrypted cookies, make calls to internal authentication and authorization services, and even interact with third-party APIs if granted permission. Permissions provide a transparent account of exactly what operations the plugins can perform. The sandbox ensures the plugins do not exceed their granted authority.

Data insight and granted permissions make Bulwark very effective for hosting anti-fraud and business logic security functions. Because you can deploy Bulwark at network ingresses, it can protect interior services from high-volume fraud activity that might otherwise overwhelm anti-fraud systems embedded within the service applications.





