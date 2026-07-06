# CI commit-image evidence

This file intentionally lives in the `media/` module so the multibranch CI
pipeline detects a service change and proves that the resulting Docker image is
tagged with the branch head commit ID.
