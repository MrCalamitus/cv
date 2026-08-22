
**endPoint : https://api.github.com/graphql**  
Query:
````
query User {
    user(login: "MrCalamitus") {
        commitComments {
            totalCount
        }
    year2026: contributionsCollection(from: "2026-01-01T00:00:00Z", to: "2026-12-31T23:59:59Z") {
      # Lo que ya tenías:
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2025: contributionsCollection(from: "2025-01-01T00:00:00Z", to: "2025-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2024: contributionsCollection(from: "2024-01-01T00:00:00Z", to: "2024-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2023: contributionsCollection(from: "2023-01-01T00:00:00Z", to: "2023-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2022: contributionsCollection(from: "2022-01-01T00:00:00Z", to: "2022-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2021: contributionsCollection(from: "2021-01-01T00:00:00Z", to: "2021-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2020: contributionsCollection(from: "2020-01-01T00:00:00Z", to: "2020-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2019: contributionsCollection(from: "2019-01-01T00:00:00Z", to: "2019-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2018: contributionsCollection(from: "2018-01-01T00:00:00Z", to: "2018-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2017: contributionsCollection(from: "2017-01-01T00:00:00Z", to: "2017-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2016: contributionsCollection(from: "2016-01-01T00:00:00Z", to: "2016-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2015: contributionsCollection(from: "2015-01-01T00:00:00Z", to: "2015-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
    year2014: contributionsCollection(from: "2014-01-01T00:00:00Z", to: "2014-12-31T23:59:59Z") {
      # Lo que ya tenías:
      totalCommitContributions
      restrictedContributionsCount
      
      # Las nuevas métricas de liderazgo y colaboración:
      totalPullRequestContributions
      totalPullRequestReviewContributions
      totalIssueContributions
      totalRepositoriesWithContributedCommits
      
      # (Opcional) Desglose específico por repositorio en ese año
      commitContributionsByRepository(maxRepositories: 5) {
        repository {
          name
          isPrivate
        }
        contributions(first: 1) {
          totalCount
        }
      }
    }
        repositories(first: 100, ownerAffiliations: OWNER, orderBy: {field: STARGAZERS, direction: DESC}) {
            totalCount
            totalDiskUsage # El peso total de tu código (en kilobytes)
            nodes {
                name
                createdAt
                isPrivate # Permite contar cuántos proyectos privados mantienes vs públicos
            }
        }
        bio
        company
        followers {
            totalCount  
        }
        pinnedItems(first: 20, types: REPOSITORY) {
            nodes {
                ... on Repository {
                name
                description
                stargazerCount
                forkCount
                primaryLanguage {
                    name
                }
                }
            }
        }
        
    }
}
````

Response :   
````
{
    "data": {
        "user": {
            "commitComments": {
                "totalCount": 20
            },
            "year2026": {
                "totalCommitContributions": 248,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 18,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 0,
                "totalRepositoriesWithContributedCommits": 11,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "certisep",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 65
                        }
                    },
                    {
                        "repository": {
                            "name": "certisep-prospeccion",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 45
                        }
                    },
                    {
                        "repository": {
                            "name": "hueleAGas",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 41
                        }
                    },
                    {
                        "repository": {
                            "name": "Personal",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 27
                        }
                    },
                    {
                        "repository": {
                            "name": "photographOS",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 25
                        }
                    }
                ]
            },
            "year2025": {
                "totalCommitContributions": 644,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 143,
                "totalPullRequestReviewContributions": 12,
                "totalIssueContributions": 4,
                "totalRepositoriesWithContributedCommits": 8,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "certisep",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 286
                        }
                    },
                    {
                        "repository": {
                            "name": "COFEPRIS",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 240
                        }
                    },
                    {
                        "repository": {
                            "name": "alumni",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 93
                        }
                    },
                    {
                        "repository": {
                            "name": "e.firma",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 8
                        }
                    },
                    {
                        "repository": {
                            "name": "Centinela",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 8
                        }
                    }
                ]
            },
            "year2024": {
                "totalCommitContributions": 1683,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 605,
                "totalPullRequestReviewContributions": 153,
                "totalIssueContributions": 71,
                "totalRepositoriesWithContributedCommits": 8,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "COFEPRIS",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 1407
                        }
                    },
                    {
                        "repository": {
                            "name": "pimatina",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 171
                        }
                    },
                    {
                        "repository": {
                            "name": "actasPREP",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 70
                        }
                    },
                    {
                        "repository": {
                            "name": "certisep",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 16
                        }
                    },
                    {
                        "repository": {
                            "name": "e.firma",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 13
                        }
                    }
                ]
            },
            "year2023": {
                "totalCommitContributions": 1608,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 599,
                "totalPullRequestReviewContributions": 1,
                "totalIssueContributions": 94,
                "totalRepositoriesWithContributedCommits": 11,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "COFEPRIS",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 1201
                        }
                    },
                    {
                        "repository": {
                            "name": "pimatina",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 215
                        }
                    },
                    {
                        "repository": {
                            "name": "actasPREP",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 71
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 70
                        }
                    },
                    {
                        "repository": {
                            "name": "e.firma",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 25
                        }
                    }
                ]
            },
            "year2022": {
                "totalCommitContributions": 333,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 5,
                "totalPullRequestReviewContributions": 2,
                "totalIssueContributions": 0,
                "totalRepositoriesWithContributedCommits": 14,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 135
                        }
                    },
                    {
                        "repository": {
                            "name": "actasPREP",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 63
                        }
                    },
                    {
                        "repository": {
                            "name": "certisep",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 54
                        }
                    },
                    {
                        "repository": {
                            "name": "SIVEI",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 38
                        }
                    },
                    {
                        "repository": {
                            "name": "enphoque",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 20
                        }
                    }
                ]
            },
            "year2021": {
                "totalCommitContributions": 580,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 27,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 1,
                "totalRepositoriesWithContributedCommits": 8,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "rin",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 249
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 148
                        }
                    },
                    {
                        "repository": {
                            "name": "certisep",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 117
                        }
                    },
                    {
                        "repository": {
                            "name": "taller-aws2021",
                            "isPrivate": false
                        },
                        "contributions": {
                            "totalCount": 21
                        }
                    },
                    {
                        "repository": {
                            "name": "validationPREP",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 19
                        }
                    }
                ]
            },
            "year2020": {
                "totalCommitContributions": 771,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 55,
                "totalPullRequestReviewContributions": 1,
                "totalIssueContributions": 202,
                "totalRepositoriesWithContributedCommits": 6,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "rin",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 715
                        }
                    },
                    {
                        "repository": {
                            "name": "certisep",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 41
                        }
                    },
                    {
                        "repository": {
                            "name": "appGolf",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 7
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 4
                        }
                    },
                    {
                        "repository": {
                            "name": "enlace-visual",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 3
                        }
                    }
                ]
            },
            "year2019": {
                "totalCommitContributions": 767,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 140,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 158,
                "totalRepositoriesWithContributedCommits": 6,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "rin",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 581
                        }
                    },
                    {
                        "repository": {
                            "name": "certisep",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 122
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 55
                        }
                    },
                    {
                        "repository": {
                            "name": "certisepAPP",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 4
                        }
                    },
                    {
                        "repository": {
                            "name": "secureSign",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 3
                        }
                    }
                ]
            },
            "year2018": {
                "totalCommitContributions": 13,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 9,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 46,
                "totalRepositoriesWithContributedCommits": 2,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "rideUpp",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 9
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 4
                        }
                    }
                ]
            },
            "year2017": {
                "totalCommitContributions": 73,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 64,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 173,
                "totalRepositoriesWithContributedCommits": 5,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "rideUpp",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 28
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 18
                        }
                    },
                    {
                        "repository": {
                            "name": "upperbus",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 15
                        }
                    },
                    {
                        "repository": {
                            "name": "appGolf",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 11
                        }
                    },
                    {
                        "repository": {
                            "name": "policeShields",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 1
                        }
                    }
                ]
            },
            "year2016": {
                "totalCommitContributions": 99,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 80,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 16,
                "totalRepositoriesWithContributedCommits": 5,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "upperbus",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 70
                        }
                    },
                    {
                        "repository": {
                            "name": "softvWeb",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 20
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 5
                        }
                    },
                    {
                        "repository": {
                            "name": "policeShields",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 3
                        }
                    },
                    {
                        "repository": {
                            "name": "Merik",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 1
                        }
                    }
                ]
            },
            "year2015": {
                "totalCommitContributions": 81,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 17,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 0,
                "totalRepositoriesWithContributedCommits": 7,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "merikBack",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 25
                        }
                    },
                    {
                        "repository": {
                            "name": "madema",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 23
                        }
                    },
                    {
                        "repository": {
                            "name": "Porcioname",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 15
                        }
                    },
                    {
                        "repository": {
                            "name": "functionsJS",
                            "isPrivate": false
                        },
                        "contributions": {
                            "totalCount": 8
                        }
                    },
                    {
                        "repository": {
                            "name": "capturaTarjetas",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 5
                        }
                    }
                ]
            },
            "year2014": {
                "totalCommitContributions": 723,
                "restrictedContributionsCount": 0,
                "totalPullRequestContributions": 4,
                "totalPullRequestReviewContributions": 0,
                "totalIssueContributions": 1,
                "totalRepositoriesWithContributedCommits": 2,
                "commitContributionsByRepository": [
                    {
                        "repository": {
                            "name": "Merik",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 602
                        }
                    },
                    {
                        "repository": {
                            "name": "capturaTarjetas",
                            "isPrivate": true
                        },
                        "contributions": {
                            "totalCount": 121
                        }
                    }
                ]
            },
            "repositories": {
                "totalCount": 48,
                "totalDiskUsage": 1863287,
                "nodes": [
                    {
                        "name": "AI4Devs-intro",
                        "createdAt": "2024-10-10T23:33:09Z",
                        "isPrivate": false
                    },
                    {
                        "name": "cv",
                        "createdAt": "2026-08-22T01:00:51Z",
                        "isPrivate": false
                    },
                    {
                        "name": "asistify",
                        "createdAt": "2026-08-14T19:22:54Z",
                        "isPrivate": true
                    },
                    {
                        "name": "MatchAuto",
                        "createdAt": "2026-08-07T18:01:22Z",
                        "isPrivate": true
                    },
                    {
                        "name": "lucaDonation",
                        "createdAt": "2026-06-02T01:34:41Z",
                        "isPrivate": true
                    },
                    {
                        "name": "Personal",
                        "createdAt": "2026-05-04T04:02:51Z",
                        "isPrivate": true
                    },
                    {
                        "name": "photographOS",
                        "createdAt": "2026-04-05T14:59:41Z",
                        "isPrivate": true
                    },
                    {
                        "name": "cotizaciones",
                        "createdAt": "2026-01-15T21:29:53Z",
                        "isPrivate": true
                    },
                    {
                        "name": "horarios",
                        "createdAt": "2025-08-04T23:49:31Z",
                        "isPrivate": true
                    },
                    {
                        "name": "alumni",
                        "createdAt": "2025-04-18T19:11:35Z",
                        "isPrivate": true
                    },
                    {
                        "name": "certisep",
                        "createdAt": "2025-02-05T01:41:21Z",
                        "isPrivate": true
                    },
                    {
                        "name": "AI4Devs-finalproject",
                        "createdAt": "2025-02-04T22:04:45Z",
                        "isPrivate": false
                    },
                    {
                        "name": "AI4Devs-lab-ides",
                        "createdAt": "2024-11-11T02:31:39Z",
                        "isPrivate": false
                    },
                    {
                        "name": "AI4Devs-stopwatch",
                        "createdAt": "2024-10-21T20:35:13Z",
                        "isPrivate": false
                    },
                    {
                        "name": "mrcalamitus.github.io",
                        "createdAt": "2023-04-19T14:02:29Z",
                        "isPrivate": false
                    },
                    {
                        "name": "softwareValidate",
                        "createdAt": "2022-10-11T01:42:18Z",
                        "isPrivate": true
                    },
                    {
                        "name": "frontenis",
                        "createdAt": "2022-09-07T18:14:02Z",
                        "isPrivate": true
                    },
                    {
                        "name": "network-actas-prep",
                        "createdAt": "2022-08-16T14:43:27Z",
                        "isPrivate": true
                    },
                    {
                        "name": "mrcalamitus",
                        "createdAt": "2022-08-03T17:41:36Z",
                        "isPrivate": false
                    },
                    {
                        "name": "verificationSystem",
                        "createdAt": "2022-05-18T14:39:43Z",
                        "isPrivate": false
                    },
                    {
                        "name": "SIVEI",
                        "createdAt": "2022-02-22T19:41:48Z",
                        "isPrivate": true
                    },
                    {
                        "name": "worker-loader",
                        "createdAt": "2021-11-17T00:59:02Z",
                        "isPrivate": false
                    },
                    {
                        "name": "joedicastro.com",
                        "createdAt": "2021-09-08T23:39:04Z",
                        "isPrivate": false
                    },
                    {
                        "name": "PrepIEEMValidation",
                        "createdAt": "2021-05-13T19:50:10Z",
                        "isPrivate": true
                    },
                    {
                        "name": "enlace-visual",
                        "createdAt": "2020-06-18T14:30:54Z",
                        "isPrivate": true
                    },
                    {
                        "name": "hyperblog",
                        "createdAt": "2020-06-02T14:26:02Z",
                        "isPrivate": false
                    },
                    {
                        "name": "SAVuegram",
                        "createdAt": "2020-01-08T13:32:22Z",
                        "isPrivate": false
                    },
                    {
                        "name": "lambda-serverless",
                        "createdAt": "2019-12-15T04:43:33Z",
                        "isPrivate": false
                    },
                    {
                        "name": "madema",
                        "createdAt": "2019-12-08T20:02:47Z",
                        "isPrivate": true
                    },
                    {
                        "name": "mapbox-gl-native",
                        "createdAt": "2019-03-05T15:44:19Z",
                        "isPrivate": false
                    },
                    {
                        "name": "pruebaCodePipeLine",
                        "createdAt": "2019-01-22T03:42:36Z",
                        "isPrivate": false
                    },
                    {
                        "name": "PayPal-Cordova-Plugin",
                        "createdAt": "2018-11-21T16:44:26Z",
                        "isPrivate": false
                    },
                    {
                        "name": "PhotoSwipe",
                        "createdAt": "2018-11-20T01:33:10Z",
                        "isPrivate": false
                    },
                    {
                        "name": "framework7-plugin-keypad",
                        "createdAt": "2018-02-05T21:39:04Z",
                        "isPrivate": false
                    },
                    {
                        "name": "angular-lazy-load",
                        "createdAt": "2018-01-05T22:01:09Z",
                        "isPrivate": false
                    },
                    {
                        "name": "Font-Awesome",
                        "createdAt": "2017-12-17T04:50:12Z",
                        "isPrivate": false
                    },
                    {
                        "name": "cordova-plugin-googlemaps",
                        "createdAt": "2017-07-15T02:51:44Z",
                        "isPrivate": false
                    },
                    {
                        "name": "cordova-plugin-googlemaps-doc",
                        "createdAt": "2017-05-06T06:25:01Z",
                        "isPrivate": false
                    },
                    {
                        "name": "checklist-model",
                        "createdAt": "2016-06-06T13:13:51Z",
                        "isPrivate": false
                    },
                    {
                        "name": "ui-select2",
                        "createdAt": "2016-03-03T01:52:19Z",
                        "isPrivate": false
                    },
                    {
                        "name": "ui-router",
                        "createdAt": "2016-01-27T19:07:00Z",
                        "isPrivate": false
                    },
                    {
                        "name": "testAngular",
                        "createdAt": "2016-01-27T16:05:56Z",
                        "isPrivate": false
                    },
                    {
                        "name": "angular.js",
                        "createdAt": "2015-07-09T23:13:05Z",
                        "isPrivate": false
                    },
                    {
                        "name": "bootstrap-datepicker",
                        "createdAt": "2015-05-26T14:19:14Z",
                        "isPrivate": false
                    },
                    {
                        "name": "functionsJS",
                        "createdAt": "2015-02-25T00:53:23Z",
                        "isPrivate": false
                    },
                    {
                        "name": "rm",
                        "createdAt": "2015-02-22T04:01:32Z",
                        "isPrivate": true
                    },
                    {
                        "name": "bootstrap",
                        "createdAt": "2014-10-25T01:05:43Z",
                        "isPrivate": false
                    },
                    {
                        "name": "jquery.jqprint",
                        "createdAt": "2014-10-09T15:21:51Z",
                        "isPrivate": false
                    }
                ]
            },
            "bio": "Founder @ZogaMx | Master in Cybersecurity | AWS Cloud Architect & AI Integration Specialist (RAG, Multi-agent systems) | 15+ years scaling digital products.",
            "company": "zoga",
            "followers": {
                "totalCount": 6
            },
            "pinnedItems": {
                "nodes": []
            }
        }
    }
}
````